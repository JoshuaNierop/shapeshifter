# Plan — wat we gaan bouwen en testen

Elke optimalisatie krijgt een eigen feature-branch vanaf `josh/main`. Bench-resultaten landen in `Metadocs/Testresults.md`. Volgorde is gekozen op `verwachte-impact / inspanning`-ratio.

---

## Fase 0 — Setup & baseline (eerst)

Voor we *één* optimalisatie schrijven moeten we kunnen meten.

### 0.1 — Toolchain
- [x] Rust 1.95.0 + cargo geïnstalleerd
- [x] wasm32-unknown-unknown target toegevoegd
- [~] `wasm-pack` install loopt
- [ ] `bun i` op josh/main, controleer dat `bun run build:wasm` werkt
- [ ] `bun --bun nuxt build` produceert werkende dist

### 0.2 — Bench-uitbreiding (`josh/opt-00-bench`)
Anoesj's `bench.mjs` is OK maar te beperkt voor onze vergelijking. Wij hebben nodig:

- **Uitgebreide puzzle-suite.** Niet alleen level10/18/34, maar de hele `PuzzleLibrary` filteren op `hide:false` + een paar nieuwe synthetische "evil" puzzles. Ranges: easy (<100ms), medium (1-10s), hard (>30s).
- **JSON-output per run** met `{puzzle, engine, run_index, ms_wall, ms_total_thread, throughput, solutions_count, meta...}`. Lazy schrijven naar `Metadocs/Testresults.md` of `bench-results/<date>-<engine>.json`.
- **Statistical aggregation.** Min, median, p95, max over N runs. Anoesj's huidige bench reports min — voldoende voor binnen-engine maar misleidend tussen engines met warmup-verschillen.
- **CI-modus.** `bun run bench:ci` runt alle engines op alle puzzles, output diff-friendly markdown table.
- **Correctness check.** Iedere run vergelijkt solutions-count met de baseline (josh/main). Mismatch = abort, niet stilzwijgend doorgaan.

**Output:** branch `josh/opt-00-bench` met nieuwe `wasm/solver/bench-suite.mjs` en `Metadocs/Testresults.md` template.

### 0.3 — Baseline meting
Run de uitgebreide bench op `josh/main` (= upstream multithread-branch zonder onze edits). Sla op als "baseline" rij. Alle volgende metingen vergelijken hiertegen.

---

## Fase 1 — Optimalisaties (parallel implementeerbaar)

Elke branch staat in tabel hieronder. Status: `[ ]` open, `[~]` in progress, `[!]` blocked, `[x]` done.

### 1.A — Transposition table met Zobrist (`josh/opt-01-zobrist`)
**Wat.** Fixed-size open-addressing hash (default 2^24 entries × 16 bytes = 256 MB optie, fallback 2^20 = 16 MB). Zobrist incremental update. Per recursie-niveau: lookup `(remaining_mask, board_hash)` voor `unsolvable_subtree` flag, skip indien hit.

**Waar.** `wasm/solver/src/lib.rs` — nieuwe `TranspositionTable` struct, ge-injecteerd in `IterCtx`. Update in `apply()`/`revert()`. Lookup voor het influence-bound check.

**Verwachting.** Op puzzles met veel symmetrische sub-stukken (level10 met 5× T-stuk) factor 2-10×. Op asymmetrische (level34) misschien factor 1.1-1.5×. Risk: trashing als hash slecht is.

**Acceptance.** Geen regressie op solutions-count. Min ≥10% sneller op level10. Geen meer dan 5% slechter op enige andere puzzle.

### 1.B — Per-cell forward checking (`josh/opt-02-forward-check`)
**Wat.** Sterkere pruning dan globale `sum`-bound. Per cel: bereken `max_remaining_influence_on_cell` precomputed (cellsInfluenced × position-touches-cell matrix). Bij elke node: per cel `transforms_needed_at_cell > max_remaining_influence_on_cell` → kap.

**Waar.** `wasm/solver/src/lib.rs` — precompute `piece_touches_cell[piece][cell] = bool` en `max_influence_per_cell_remaining[depth][cell]`. Check na elke `apply()`.

**Verwachting.** Sterke pruning op moeilijke puzzles waar lokale cellen vroeg opdrogen. Verwachte factor 2-5× op level34/37.

**Acceptance.** Geen regressie. Min ≥30% sneller op level34/37.

### 1.C — MRV piece ordering (`josh/opt-03-mrv`)
**Wat.** Vervang globale `cellsInfluenced desc` sort door dynamische "kies stuk met minste valid positions". Tiebreak: cellsInfluenced desc.

**Waar.** `wasm/solver/src/lib.rs` — bij elke recursie-niveau, herbereken volgorde van remaining pieces. Lichte voor- en navarianten.

**Verwachting.** Variabel. Sommige puzzles 2-3× sneller, sommige 10% trager. Kost: extra werk per node — moet zorgvuldig.

**Acceptance.** Gemiddelde over puzzle-suite ≥15% sneller (geometrisch gemiddelde).

### 1.D — Symmetry breaking (`josh/opt-04-symmetry`)
**Wat.** Pre-process: detecteer identieke pieces (zelfde grid-data). Lex-leader constraint: `pos[P_i] <= pos[P_j]` voor identieke `P_i`, `P_j` met `i<j`.

**Waar.** `wasm/solver/src/lib.rs` — eq-class detection in `Puzzle::new()`. In `iter_placements_inner`: skip posities die de lex-orde schenden gegeven al-geplaatste identieke broers.

**Verwachting.** k!-factor reductie op puzzles met identieke stukken. Level10 (5× T) → 120× minder werk. Andere puzzles: 1×.

**Acceptance.** ≥50× sneller op level10. Geen regressie elders.

### 1.E — SIMD-128 bit-planes (`josh/opt-05-simd128`)
**Wat.** `PackedEngine` herschrijven met `std::arch::wasm32::v128` intrinsics. Twee u64-lanes → één v128 (i64x2 of i8x16 afhankelijk van opcode).

**Waar.** `wasm/solver/src/lib.rs` — `PackedEngine<BITS>` cfg-gate op `target_feature="simd128"`. Apply/revert in SIMD. Popcount via `i8x16.popcnt`.

**Verwachting.** Factor 1.5-2× op bord-cellen ≤64. Niet meer want we doen al u64 — wint alleen in cross-lane ops.

**Build flag.** `RUSTFLAGS='-C target-feature=+simd128'` in build script. Vereist Chrome 91+/FF 89+/Safari 16.4+.

**Acceptance.** ≥40% sneller op level34/37 in PackedEngine-paden.

### 1.F — Dynamic work-stealing (`josh/opt-06-work-stealing`)
**Wat.** Vervang depth-1 modulo slicing door dynamic task hand-out. Orchestrator-laag (JS) houdt task-queue; workers requesten een sub-task zodra klaar.

**Waar.** `app/composables/useParallelSolver.ts` + Rust `solve_subtree(unused_mask, board, ...)` entry. Task-granulatie: depth-2 of depth-3 subtree.

**Verwachting.** Beter load-balanced; significant op puzzles waar pruning per slice asymmetrisch is. Factor 1.5-3× over huidige multi-worker setup.

**Acceptance.** ≥20% sneller op level34 met 8 workers. Geen meer dan 5% slechter op kleine puzzles (waar slicing-overhead boven solver-cost zit).

### 1.G — Meet-in-the-Middle (`josh/opt-07-mitm`)
**Wat.** Voor puzzels met ≤8 stukken: split halverwege, enumereer beide halves, hash-meet. Detectie: trigger MITM-pad als `Π |positions|` < threshold (~2^28 voor 256 MB tabel).

**Waar.** Nieuwe `solve_mitm()` entry in Rust. Hashtable via `FxHashMap<[u8; CELLS], Vec<L_assignment>>`. Compactie: gebruik `[u64; BITS]` plane-vector als key.

**Verwachting.** Op kleine puzzels (p1-p10): 10-100× sneller. Op grote puzzels: niet gebruikt (table te groot).

**Acceptance.** Op puzzles waar trigger fires: ≥10× sneller. Geen impact op andere.

### 1.H — IDA* / branch-and-bound met betere bound (`josh/opt-08-bnb`)
**Wat.** Branch-and-bound met `h = ceil(transforms_needed_per_cell / max_per_piece)` als per-cel admissible heuristic. Maintain global lowest-cost-so-far; prune subtrees met `f > best`.

**Waar.** Re-architect `iter_placements_inner` met expliciete bound. Hier ook beter integreren met forward checking.

**Verwachting.** Onbekend. Misschien factor 1.2-2× of geen verschil.

**Acceptance.** ≥10% sneller op gem. Geen regressie >5%.

### 1.I — SAT-oracle (`josh/opt-09-sat-oracle`) — alleen voor correctness
**Wat.** Z3-pad via `bun run bench:sat`. Niet productie, alleen om elke variant tegen Z3 te vergelijken op kleine puzzles.

**Waar.** Apart Node-script in `scripts/sat-oracle.mjs` met `npm install z3-solver`. Input = PuzzleOptions; output = bool + solution.

**Acceptance.** Voor elke puzzel in suite ≤10 stukken: Z3 vindt dezelfde solutions-count als onze solver. Mismatch op enige variant = bug.

### 1.J — Native AVX2 bench-only (`josh/opt-10-native-avx2`)
**Wat.** `cargo build --release --features avx2 --target x86_64-pc-windows-msvc` met `std::arch::x86_64::__m256i` PackedEngine. Apart bin-target in `wasm/solver/src/bin/native-bench.rs`.

**Doel.** Upper-bound meten voor PackedEngine zonder wasm-overhead. Geeft ons inzicht hoeveel headroom er nog is.

**Acceptance.** Gewoon meten en rapporteren — geen pass/fail.

---

## Fase 2 — Combineer winners

Nadat 1.A-1.J los gebencht zijn: combineer de top-3 winners in `josh/opt-final`. Verwacht: synergie tussen forward-checking + MRV (sterkere pruning kost minder als je sowieso minder doorrekent), tussen SIMD + work-stealing (orthogonaal). Conflicts: MITM en transposition-table delen geheugen, één moet wijken.

---

## Fase 3 — Optioneel verdere richting

### 3.A — WebGPU compute (`josh/opt-11-webgpu`)
Alleen als fase 1+2 niet genoeg opleveren. Hoge inspanning, onzekere winst.

### 3.B — Wasm-bindgen-rayon
True threads binnen wasm i.p.v. JS-workers. Vereist threading proposal + ATOMICS build target. Vanaf hier interessant.

---

## Acceptance criteria voor "succes"

We hebben succes als we ten minste **één variant** vinden die `josh/main`-baseline met factor ≥3× verslaat op een evil-level (huidige solver > 30s) **en** geen regressie >5% op enige andere puzzle in de suite.

Verder willen we een `Testresults.md` matrix die op één scrollscherm laat zien: "X variant doet op Y puzzle Z×". Dat is het deliverable voor Joshua (en Anoesj, mocht hij willen weten wat we doen).

---

## Werkwijze per branch

1. `git checkout josh/main && git checkout -b josh/opt-NN-naam`
2. Implementeer optimalisatie (Rust en/of TS/JS waar nodig)
3. `bun run build:wasm` + (zo nodig) `bun run build`
4. `bun --bun run wasm/solver/bench-suite.mjs --variant=opt-NN` → JSON-output
5. Append rij in `Metadocs/Testresults.md`
6. Commit + push naar `origin/josh/opt-NN-naam`
7. Notitie in `Metadocs/Changelog.md` met datum + 1-zin samenvatting
8. PR-style summary in commit-message (geen daadwerkelijke PR naar Anoesj)

Bij regressie: commit alsnog (negative result is ook info), markeer in Testresults.md met ❌.

## Risico's & mitigaties

| Risico | Kans | Impact | Mitigatie |
|---|---|---|---|
| Variant heeft subtle bug (verkeerde solutions-count) | medium | hoog | SAT-oracle (1.I) checkt elke variant pre-commit |
| Bench-variability te groot om verschillen te zien | medium | medium | min 7 runs, rapporteer p50/p95, gebruik geometrisch gem. over suite |
| Joshua/Ivan acc botst (gedeeld) | laag | laag | Wij werken op fork — geen concurrentie met Anoesj |
| WASM build instable (wasm-pack drama) | medium | medium | Fall-back: directe `cargo build --target wasm32-unknown-unknown` zonder wasm-pack |
| GPU/SAT pad blokt productie-pad | laag | laag | Strikt achter feature-flags |
