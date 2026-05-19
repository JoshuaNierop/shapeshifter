# Todo — Shapeshifter

Losse taken + openstaande beslissingen. Volledige plan staat in `Planning.md` en `Plan.md`.

## Openstaande beslissingen

- [ ] P1 — Project verplaatsen van `Shelf/` naar `Hobby/` of `Tools/` na fase 0 (nu blijft in Shelf zodat URL/aliassen niet stuk gaan tijdens setup)
- [ ] P2 — Bench-output committen in repo (`bench-results/*.json`) of buiten repo houden? Compromis: alleen geaggregeerde markdown-tabel in `Testresults.md` committen, raw JSON in `.gitignore`
- [ ] P2 — Z3 SAT-oracle: `z3-solver` npm pakket (langzaam) of native `z3` binary via gh action? Voor lokaal pad: npm

## Setup (P0)

- [x] P0 — Fork gemaakt op github.com/JoshuaNierop/shapeshifter
- [x] P0 — Remotes: origin = fork, upstream = Anoesj read-only
- [x] P0 — Rust 1.95.0 + wasm32-unknown-unknown target
- [~] P0 — wasm-pack installeren (cargo install loopt, fallback = official installer binary)
- [ ] P0 — `bun i` op josh/main draaien en `bun run build:wasm` testen
- [ ] P0 — Baseline-run van Anoesj's `bench.mjs` om uitgangspositie vast te leggen

## Fase 0 — Bench-uitbreiding (P0)

- [ ] P0 — Branch `josh/opt-00-bench` aanmaken vanaf josh/main
- [ ] P0 — Volledige puzzle-suite uit `PuzzleLibrary` opnemen (filter `hide:false`)
- [ ] P0 — Bench-output naar JSON met `{puzzle, engine, run_index, ms_wall, throughput, solutions_count}`
- [ ] P1 — Statistical aggregation: min/median/p95/max over N runs
- [ ] P1 — CI-modus: `bun run bench:ci` produceert markdown-tabel
- [ ] P1 — Auto-write `Testresults.md` met laatste resultaten per variant
- [ ] P0 — Correctness assert: solutions-count == baseline, anders abort
- [ ] P2 — Synthetische "evil" puzzles toevoegen (>30s op huidige solver)

## Fase 1 — Optimalisaties (P1 — parallel implementeerbaar)

- [ ] P1 — `josh/opt-01-zobrist` Transposition table met Zobrist hashing
- [ ] P1 — `josh/opt-02-forward-check` Per-cell forward checking
- [ ] P1 — `josh/opt-03-mrv` MRV piece ordering met cellsInfluenced tiebreak
- [ ] P1 — `josh/opt-04-symmetry` Lex-leader symmetry breaking voor identieke stukken
- [ ] P1 — `josh/opt-05-simd128` SIMD-128 v128 intrinsics in PackedEngine
- [ ] P1 — `josh/opt-06-work-stealing` Dynamic task hand-out i.p.v. modulo slicing
- [ ] P2 — `josh/opt-07-mitm` Meet-in-the-Middle voor puzzles ≤8 stukken
- [ ] P2 — `josh/opt-08-bnb` Branch-and-bound met per-cel admissible heuristic
- [ ] P2 — `josh/opt-09-sat-oracle` Z3-baseline voor correctness verificatie
- [ ] P3 — `josh/opt-10-native-avx2` Native AVX2 bench-only voor upper bound meting

## Fase 2 — Combineer (P1)

- [ ] P1 — `josh/opt-final` met top-3 winners gecombineerd
- [ ] P1 — Sanity-check op alle puzzles + Z3-cross-verify

## Fase 3 — Optioneel (P3)

- [ ] P3 — `josh/opt-11-webgpu` WebGPU compute shader exploratie
- [ ] P3 — `josh/opt-12-wasm-threads` wasm-bindgen-rayon true threading

## Docs

- [x] P0 — `Metadocs/Readme.md` met frontmatter
- [x] P0 — `Metadocs/Research.md` — methoden, papers, repos
- [x] P0 — `Metadocs/Plan.md` — implementatie-plan per branch
- [ ] P1 — `Metadocs/Testresults.md` template + eerste baseline-rij
- [ ] P2 — `Metadocs/Architecture.md` — diagram van solver-paden, engines, slicing
- [ ] P2 — Per branch een commit met "Why" boven elke optimalisatie (research-quote)

## Bugs

(geen bekend — wij hebben nog geen eigen code)

## Afgerond

- [x] P0 — Vier Anoesj-branches geïnspecteerd (main, claude/multithreading, claude/rust-wasm, sse/ws backups)
- [x] P0 — Fork-strategie + git-workflow vastgesteld
- [x] P0 — Onderzoek naar 13 optimalisatie-richtingen
