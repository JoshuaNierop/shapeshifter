# Research — Shapeshifter optimalisaties

Diepe dive in technieken, papers en open-source repos die relevant zijn voor het versnellen van een polyomino-puzzle solver. Geschreven 2026-05-19. Doel: per optimalisatie-richting weten waarom het zin heeft, welke kosten erbij horen, en welke referenties we kunnen gebruiken als we vastlopen.

---

## 0. Probleem-formalisatie

**Input.** Bord `B ∈ Z_N^{R×C}` (waarden 0..N-1, N=`figuresCount`, doel = N-1). Puzzle pieces `P_1..P_k` waar elke `P_i ∈ {0,1}^{m_i×n_i}`. Per stuk `i` kies één positie `(x_i, y_i)` zodat `P_i` past op het bord.

**Operatie.** Een stuk op positie `(x,y)` plaatsen telt `+1 mod N` op elke `B[y+r][x+c]` waar `P_i[r][c]=1`.

**Doel.** Vind een keuze van posities zodat na alle plaatsingen elke cel == N-1.

**Karakter.**
- Niet pure exact cover (overlap is toegestaan, want mod-N).
- Niet pure Lights Out: elk stuk wordt **precies één keer** geplaatst, niet vrij getoggeld. Dat breekt directe lineaire algebra over Z_N.
- Wel een **constraint satisfaction problem** (CSP) over discrete domeinen (één domein per stuk = lijst posities, output-cellen moeten allemaal == target).
- Search space = `Π_i |positions_i|`. Voor level34: 12 stukken, miljoenen tot miljarden combinaties.

**Wat we al gebruiken (multithread-branch):**
1. Influence-bound vroege exit (`completed_sum - current_sum > remaining_max_influence` → kap).
2. Volgorde stukken: meest cellen-influence eerst.
3. Dedup van situaties (gridhash + remaining-pieces-mask).
4. Corner-only enumeratie voor `preparePossibleSolutionStarts` (bitmask).
5. Mutate-and-revert i.p.v. clone.
6. Bit-plane packed board (PackedEngine, fc≤16, cells≤64).
7. `Position`-singleton voor cache-locality.
8. Depth-1 modulo work-slicing across workers.
9. SAB stop-flag voor early bail-out.

Wat nog **niet** in zit — onze speelveld.

---

## 1. Meet-in-the-Middle (MITM) op piece-splitsing

**Idee.** Splits de stukken in twee helften L = `P_1..P_{k/2}` en R = `P_{k/2+1}..P_k`. Enumereer alle plaatsingen van L → hash `(per-cell-bijdrage mod N)` → bijdrage_L. Doe hetzelfde voor R. Match `bijdrage_L + bijdrage_R ≡ target - B (mod N)` via hashtable lookup.

**Complexiteit.** O(√(brute-force)) = `O(N_L + N_R)` waar `N_L = Π |positions_i|` over L. Voor `k=12` stukken met 50 posities elk: brute force = 50^12 ≈ 2.4e20, MITM = 2 × 50^6 ≈ 3e10 = 8 ordes minder.

**Catch.**
- Geheugen: hashtable moet 50^6 = 15B entries dragen. Met R×C cells × log2(N) bits = bijv 36 × 2 bits = 72 bits ≈ 9 bytes per state → 140 GB. Niet feasible op laptop.
- Mitigatie: hash de state-vector → bucket → bewaar alleen de bucket-counter, geen volledige state. Of: bewaar alleen *gesorteerde* states zodat collision-detect kan, niet expand.
- Voor kleinere puzzels (≤8 stukken) is MITM wel feasible en kan ordes van grootte sneller zijn dan brute force.

**Refs:**
- [Beating Meet-in-the-Middle for Subset Balancing Problems (arXiv 2511.10823, 2025)](https://arxiv.org/abs/2511.10823) — recent average-case beat tot O(2^0.337n).
- [MM: A bidirectional search algorithm guaranteed to meet in the middle (Holte 2017)](https://www.sciencedirect.com/science/article/pii/S0004370217300905) — ook relevant voor IDA*-varianten.
- [Programming Praxis: Subset Sum, Meet In The Middle](https://programmingpraxis.com/2012/03/30/subset-sum-meet-in-the-middle/) — clean pseudocode.

**Implementatie-pad.** Rust-side: nieuwe `solve_mitm()` entry. Split unused pieces op `unused.len()/2`. Enumereer L: voor elke combinatie van posities (`Π |pos_L|`), bouw de board-delta vector als `[u8; cells]`, hash met FxHash, insert in `IndexMap<u64, Vec<L_assignment>>`. Loop R: bereken delta_R, compute `needed = target - delta_R - init_board (mod N)`, hash en lookup.

---

## 2. Dancing Links / Algorithm X (DLX) — *waarschijnlijk niet direct toepasbaar*

**Idee.** Knuth's klassieker voor *exact cover*. Matrix met kolommen = constraints, rijen = keuzes; selecteer een rijset zodat elke kolom precies 1× gedekt.

**Probleem.** Shapeshifter is **niet** exact cover. Cellen mogen meerdere keren "geraakt" worden (mod-N accumulatie). De decompositie `cel-constraint = som van piece-bijdragen ≡ target` is een lineair-modulair systeem, niet 0/1.

**Hybride.** Wel bruikbaar voor de sub-stap "stukken die elkaar niet mogen overlappen" indien we N=2 met enkele plaatsing zouden hebben. In ons geval valt dat weg.

**Verdict.** Overslaan — zou een herformulering vereisen die de mod-N structuur breekt. Wel benoemd in research voor de volledigheid en omdat alle pentomino-papers ermee beginnen.

**Refs:** [Knuth, Dancing Links (arXiv cs/0011047)](https://arxiv.org/abs/cs/0011047), [SageMath TilingSolver](https://doc.sagemath.org/html/en/reference/combinat/sage/combinat/tiling.html), [jwg4/polyomino](https://github.com/jwg4/polyomino).

---

## 3. Transposition table + Zobrist-style hashing

**Idee.** Sla `(remaining_pieces_bitmask, board_state)` op met `(known_unsolvable | known_partial_value)`. Bij hertegenkomst kunnen we direct skippen. Anoesj had dit in TS (`#uniqueSituations`) maar zette het uit wegens memory; in Rust met fixed-size open-addressing kunnen we het wel.

**Zobrist.** Random u64 per (cel × N-waarde) en per (piece × position). Hash = XOR. Update bij plaatsing = `hash ^= zobrist[piece][pos]; for each touched cell: hash ^= cell_old; hash ^= cell_new`. **O(touched_cells) incrementeel** — geen volledige rehash per node.

**Tabelstructuur.** Open-addressing, fixed 2^N entries (bijv 2^24 = 16M, ~256 MB). Entry = `(hash_high, depth, flag)` waar flag ∈ {unsolvable_subtree, partial, exact}. Eviction: depth-preferred (diepere zoekresultaten zijn waardevoller).

**Combineren met multithread.** Per-worker tabel of shared? Per-worker is veiliger (geen sync), shared kost atomic en is gevoelig voor cache-thrashing. Standaard: per-worker.

**Refs:**
- [Chessprogramming wiki — Zobrist Hashing](https://www.chessprogramming.org/Zobrist_Hashing) — definitie + incrementele update.
- [Chessprogramming wiki — Transposition Table](https://www.chessprogramming.org/Transposition_Table) — replacement schemes.
- [Wikipedia: Transposition table](https://en.wikipedia.org/wiki/Transposition_table) — memoization-as-DP framing.

---

## 4. Smartere zoekvolgorde (MRV / fail-first)

**Idee.** Op elk niveau van de recursie: kies het *meest beperkte* stuk eerst (Minimum Remaining Values heuristic uit CSP). "Beperkt" = laagste aantal geldige posities gegeven huidige bord-state.

**Huidige stand.** Stukken worden gesorteerd op `cellsInfluenced desc` — dat is **maximum impact**, niet fail-first. Werkt soms maar mist constraints.

**Hybride.** Gebruik MRV als primair, cells-influenced als tiebreak. Voor stukken met << posities (bijv. spans-X-axis stukken in een smal bord) wordt het effect groot.

**Sub-idee — position ordering.** Binnen één stuk: sorteer posities op `(transforms_needed_after_placement, distance_to_focus_area)`. Anoesj heeft focus-area logica in `GameBoard.analyze()` zitten maar gebruikt het niet (commented-out in `#puzzlePiecePlacementOptionsIterator`). Reden volgens commit: kosten > baten op gemeten levels.

**Refs:** [AIMA chap 5 — CSP](http://aima.cs.berkeley.edu/newchap05.pdf) — section "Variable and value ordering" (MRV, LCV, degree heuristic).

---

## 5. Forward checking / constraint propagation

**Idee.** Na elke plaatsing: loop door alle nog-niet-geplaatste stukken; voor elk stuk, filter zijn `possiblePositions` op posities die het nog mogelijk maken om de target te halen. Als één domein leeg wordt → backtrack onmiddellijk.

**Toepassing op Shapeshifter.** Een positie `pos` is "nog mogelijk" als — gegeven de huidige bord-state — er een **legaal eindbord** bestaat dat door deze plaatsing + iets-van-de-rest bereikbaar is. Volledige test is zelf NP-hard. Zwakke maar snelle test:
- Per cel: kan de som van remaining-pieces deze cel naar target draaien? (Per cel: hoeveel times kan het nog +1'd worden door remaining pieces × hun posities die deze cel raken?)
- Als één cel `transforms_needed > remaining_max_at_cell` → onmogelijk.

Dit is een **per-cell** versie van de huidige globale `sum`-bound, en strikt sterker.

**Verwacht effect.** Significant op grote boards (level34/35) waar lokale cellen vroeg "opdrogen" qua remaining-influence. Kost: O(cells × remaining_pieces × positions_per_piece) per call. Throttelen met `iter_check_counter` zoals nu bij time-check.

**Refs:**
- [AC-3 algoritme, AIMA](http://aima.cs.berkeley.edu/newchap05.pdf).
- [Constraint propagation overzicht — Bartak](https://ktiml.mff.cuni.cz/~bartak/constraints/propagation.html).
- [Sandipan: Sudoku met arc-consistency + forward checking](https://sandipanweb.wordpress.com/2017/03/17/solving-sudoku-as-a-constraint-satisfaction-problem-using-constraint-propagation-with-arc-consistency-checking-and-then-backtracking-with-minimum-remaning-value-heuristic-and-forward-checking/).

---

## 6. Work-stealing parallelisme (i.p.v. modulo slicing)

**Idee.** Huidige aanpak: depth-1 modulo slicing → elke worker enumereert dezelfde tree, neemt alleen taken met `task_idx % N == worker_id`. Probleem: workers die snel klaar zijn (omdat hun slice veel kan prunen) wachten op trage workers (slechte load balancing).

**Work-stealing.** Elke worker heeft een eigen deque met sub-tasks. Wanneer idle, steel uit andermans deque. Klassieke Cilk/Rayon design.

**WASM-context.** Geen Rayon binnen wasm-bindgen (geen threads in standaard wasm). Wel via:
- **`wasm-bindgen-rayon`** — vereist `wasm32-unknown-unknown` + threading proposal + ATOMICS. Werkt achter COOP/COEP.
- **Eigen JS-laag work stealing** — orchestrator-laag (`useParallelSolver.ts`) houdt een gedeelde task-queue in SharedArrayBuffer; workers polynomiale heen-en-weer kost de winst weg op fijne tasks.

**Pragmatisch.** Begin met **dynamische task-uitdeling** in JS-laag: workers requesten een nieuwe sub-task via `postMessage` zodra ze klaar zijn. Granulatie: depth-2 of depth-3 tasks (kleiner dan depth-1 → meer parallelisme, meer overhead).

**Refs:**
- [Rayon work-stealing](https://docs.rs/rayon/latest/rayon/fn.join.html).
- [Rust workload 10x faster (with/without Rayon) — Guillaume Endignoux](https://gendignoux.com/blog/2024/11/18/rust-rayon-optimized.html).
- [Software Patterns: Work Stealing](https://softwarepatternslexicon.com/rust/concurrency-and-parallelism-in-rust/work-stealing-and-task-scheduling-patterns/).

---

## 7. SIMD-128 bit-plane operations (wasm32 v128)

**Idee.** Huidige PackedEngine doet ripple-carry over `[u64; BITS]` planes. SIMD-128 (v128) doet hetzelfde over 128 bits per instructie = factor 2 op gebufferd werk, soms meer met intelligente instructies.

**Wins op huidige hot path.**
- `apply()` ripple-carry: XOR + AND in 128-bit lanes (2 cellen-banks tegelijk).
- `match_v` AND-of-planes: idem.
- `popcount(mask & match)`: v128 heeft `i8x16.popcnt` (1 instr).

**Kosten.** Bord moet `cells <= 128` of we splitsen in twee `u64x2`s. Voor level34 (6×6=36 cells) trivial. Voor level37 (6×6 = 36) ook. Pas bij borden ≥9×9 begint het te steken.

**Implementatie.** Rust `std::arch::wasm32::v128` intrinsics + cfg-gate op `target_feature = "simd128"`. Cargo target features: `wasm32-unknown-unknown` met `-C target-feature=+simd128`. Browser support: alle moderne (Chrome 91+, FF 89+, Safari 16.4+).

**Refs:**
- [V8 SIMD intro](https://v8.dev/features/simd).
- [WebAssembly SIMD spec proposals](https://github.com/WebAssembly/simd/blob/main/proposals/simd/SIMD.md).
- [MDN: v128 type](https://developer.mozilla.org/en-US/docs/WebAssembly/Reference/Types/v128).

---

## 8. Symmetry breaking — identieke stukken

**Idee.** Als twee stukken `P_i` en `P_j` exact dezelfde grid hebben (zelfde shape, zelfde cellsInfluenced, zelfde positions-set), dan zijn elke twee oplossingen die `P_i` en `P_j` swappen identiek. We zoeken nu beide → factor 2 (of k!) verspilling.

**Fix.** Canonicaal lex-leader: forceer dat in elke oplossing `P_i.position <= P_j.position` voor alle paren van identieke stukken (lexicografisch). Restricteer tijdens enumeratie: zodra `P_i` op positie `p_i` ligt, mag `P_j` alleen op `p_j >= p_i`.

**Detect.** Pre-process: groepeer stukken op `hash(grid)` → equivalence classes → genereer ordering constraints.

**Verwacht effect.** Sterk afhankelijk van puzzle. Level10 heeft veel `[[1,0],[1,1],[1,0]]`-stukken (T-vorm) → factor 5! = 120 of meer reductie. Andere levels weinig of niet.

**Refs:**
- [Symmetry Breaking in Constraint Satisfaction (research)](https://www.researchgate.net/publication/2522587_Symmetry_Breaking_in_Constraint_Satisfaction).
- [Breaking Symmetry with Different Orderings (arXiv 1306.5053)](https://arxiv.org/abs/1306.5053).

---

## 9. IDA* / Iterative Deepening met heuristiek

**Idee.** Brute-force is in feite DFS-tot-leaf. IDA* doet DFS met threshold `f = g + h` waar `g` = aantal stukken geplaatst, `h` = admissible lower bound op resterende kosten. Threshold start laag, verhoogt iteratief.

**Heuristic in Shapeshifter.** `h = ceil(transforms_needed / max_influence_per_remaining_piece)`. Dat is admissible (kun je niet doen met minder stukken). Probleem: alle stukken **moeten** gebruikt worden, dus `g+h` heeft een fixed maximum bij `g=k` (alle stukken op). IDA* wordt dan eigenlijk "branch-and-bound met sorted node expansion".

**Bruikbaarder framing — Best-First / A***. Maintain priority queue van partial solutions geordend op `h(remaining)`. Pop hoogst, expand. Vraag: memory exposure (priority queue groeit). Geen go-to.

**Verdict.** Geen makkelijke win zonder een veel betere `h`. Wel een vervolg-richting als forward-checking goed werkt.

**Refs:**
- [Korf, Depth-First Iterative-Deepening (1985)](https://www.cse.sc.edu/~mgv/csce580f09/gradPres/korf_IDAStar_1985.pdf) — canonical.
- [Wikipedia: IDA*](https://en.wikipedia.org/wiki/Iterative_deepening_A*).

---

## 10. SAT / SMT reductie

**Idee.** Formuleer Shapeshifter als CNF / SMT (bv. Z3). Variabelen: `x_{i,p}` = "stuk i op positie p". Constraints:
- Exactly-one: voor elk stuk i, precies één p actief.
- Per cel: `Σ_{i,p : touches} x_{i,p} ≡ target - B[cell] (mod N)`.

Voor N=2 is dit pure CNF (XOR constraints), oplosbaar met **Gauss-elimination + DPLL**. Voor N=3 cumulatief lastiger; SMT-LIB met `(Int)` types werkt maar performt onvoorspelbaar.

**Verwachting.** Voor kleine puzzels (≤8 stukken, ≤6×6): Z3 in milliseconds. Voor grote (level37, 15 stukken, 6×6): Z3 kan minuten worden, want NP-hard. Onze handgemaakte solver met domain-specifieke pruning verslaat een generic SAT/SMT solver vaak.

**Waarde.** Vooral als **referentie-oracle** (vergelijkbare antwoord, verificatie van eigen solver). Niet als productie-pad.

**Refs:**
- [Tile-laying Puzzle Solver via SAT (huynd2210)](https://github.com/huynd2210/Tile-laying-Puzzle-Solver).
- [Z3 voor SAT/SMT puzzles (UMD CMSC433)](https://www.cs.umd.edu/class/fall2025/cmsc433/Solving_SAT_and_SMT_Problems_Using_Z3.html).
- [SAT/SMT on Sudoku eval (arXiv 2501.08569)](https://arxiv.org/abs/2501.08569).

---

## 11. GPU / WebGPU compute shader

**Idee.** WebGPU compute shader → duizenden threads parallel. Elke thread = één partiële oplossing-tak.

**Realiteit.**
- Bord-state past in 32-byte buffer (level34: 36 cells × 1 byte). Per-thread state-stack voor recursie kost ~kilobyte. 1000 threads = 1 MB buffer. Goed te doen.
- Probleem 1: **divergente flow**. GPU SIMT lanes haten branching, en brute force is **all branching**. Verwachte efficiency: ~10-20% van peak.
- Probleem 2: hashtable / dedup is bijna onmogelijk efficient op GPU.
- Probleem 3: Browser support — Chrome 113+/Safari 18+. Mobiel nog wankel.

**Verwachting.** Misschien 2-5× over CPU multithread voor *brute force zonder dedup*. Niet de moeite voor de eerste iteratie. Als rest van het stack staat → laatste richting.

**Refs:**
- [V8 WebGPU compute basics](https://webgpufundamentals.org/webgpu/lessons/webgpu-compute-shaders.html).
- [WebGPU compute showcase (scttfrdmn)](https://github.com/scttfrdmn/webgpu-compute-exploration).

---

## 12. SIMD-256 / cross-lane: AVX2 via native binary

**Idee.** Voor benchmark-doeleinden: een native Rust binary (geen wasm) met AVX2/AVX-512 over u256/u512 planes. Dit is **niet** de productie-doel (browser-app), maar geeft ons een **upper bound** waar de WASM-versie naartoe kan groeien.

**Toepassing.** `wasm/solver/bench.mjs` is al een Bun bench. We kunnen een tweede native bench-target maken (`cargo build --release` → run direct).

---

## 13. Pattern databases / precomputed sub-puzzle solutions

**Idee.** Voor kleine subset van puzzles (eerste 3 stukken vast): precompute alle eindboord-states. Bij real solve: opzoeken in DB.

**Probleem.** State space groeit exponentieel met stukken; DB voor 4+ stukken al multi-GB. Niet praktisch zonder hash-compressie.

**Verdict.** Skip. Wel benoemd voor de volledigheid.

---

## Belangrijke referentie-repos

| Repo | Taal | Wat | Wat we ervan kunnen overnemen |
|---|---|---|---|
| [stdio2016/polysolve](https://github.com/stdio2016/polysolve) | C++ + OpenMP | Multi-grid polyform solver | `--parallel-level` argument: depth-N task generation als basis voor dynamic work stealing |
| [grego/rust-polyomino](https://github.com/grego/rust-polyomino) | Rust | DLX-gebaseerd, multithread via thread-clone | Performance baseline (~68ms voor 20×3 pentomino) |
| [detrin/rust-sat-polyomino](https://github.com/detrin/rust-sat-polyomino) | Rust + varisat | SAT-reductie | Referentie-oracle voor correctness |
| [cemulate/polyomino-solver](https://github.com/cemulate/polyomino-solver) | TS (browser) | DLX in JS | UI-design ideeën |
| [HanzFelix/polyomino-solver](https://github.com/HanzFelix/polyomino-solver) | JS | Browser-based, exhaustive | Niet relevant qua perf |
| [itchyny/bitboard](https://github.com/itchyny/bitboard) | Rust | Bitboard library | API-design voor onze packed engine |

## Kern-papers (priority-reading)

1. [Knuth — Dancing Links](https://arxiv.org/abs/cs/0011047) — niet direct toepasbaar maar verplicht voor context.
2. [Korf — Depth-First Iterative Deepening (1985)](https://www.cse.sc.edu/~mgv/csce580f09/gradPres/korf_IDAStar_1985.pdf) — IDA* canonical.
3. [Holte et al — MM: bidirectional search guaranteed to meet in the middle](https://www.cs.du.edu/~sturtevant/papers/MMaaai.pdf) — als we MITM doorzetten.
4. [Adaptive Parallel Iterative Deepening Search](https://www.researchgate.net/publication/51893578_Adaptive_Parallel_Iterative_Deepening_Search) — voor work-stealing analyse.
5. [Lights Out in p Colors (arXiv 2510.23677, 2025)](https://arxiv.org/abs/2510.23677) — algebraïsche structuur als achtergrond.

## Belangrijke negatieve resultaten uit Anoesj's commits

- `structuredClone` voor board state was killer. Mutate-and-revert is een ~12× winst (commit `fbed74a`).
- Async generators vs sync = 8% trager.
- Promises overal = trager.
- Saving every unique situation in een `Set<string>` = OOM op grote puzzels.
- Corner-only preparation is in veel gevallen langzamer dan direct brute forcen (`preparePossibleSolutionStarts` default off).

Dit betekent: **eerst altijd mutate-and-revert, geen async, geen promises in hot loop, dedup alleen via fixed-size hashtable.** Anders zit je achter de kracht aan.

## Open vragen voor de implementatie

1. **Bench-suite uitbreiden** met levels die de huidige solver lang draaien. Level34/35/37 zijn moeilijk; we hebben minstens één bewust "evil" level nodig om kleine verschillen meetbaar te maken.
2. **Wall-clock vs throughput** — bench moet beide rapporteren. Throughput (combs/sec) is consistenter; wall-clock is wat de gebruiker voelt.
3. **Correctness oracle** — Z3-baseline of TS-solver-baseline om elke nieuwe variant tegen te verifiëren.
4. **CI / regression detection** — wanneer een variant 10× sneller op level10 maar 2× trager op level34, willen we dat zien.
