# Testresults — Shapeshifter

Bench-resultaten en correctness-checks per variant. Eenheid: wall-clock ms (lager = beter), median van 7 runs tenzij anders aangegeven. Hardware: Joshua's laptop (RTX 3060 Laptop, 8GB RAM, x86_64-pc-windows-msvc). Browser bench-resultaten met cold/warm worker pool apart benoemen.

Tabel-structuur — per puzzel een rij, per variant een kolom. Ratio = `baseline / variant` (>1 = sneller dan baseline).

## Baseline definitie

**baseline** = `josh/main` (= upstream `claude/puzzle-solver-multithreading-option-2`) zonder eigen edits. Single-thread wasm via `bench.mjs`. Te meten zodra fase 0 klaar is.

## Smoke tests

| Check | Status | Datum | Notities |
|---|---|---|---|
| `bun i` op josh/main | ⏳ | - | nog uit te voeren |
| `bun run build:wasm` op josh/main | ⏳ | - | nog uit te voeren |
| `bun --bun nuxt build` produces dist | ⏳ | - | nog uit te voeren |
| `bun --bun run wasm/solver/bench.mjs level34 3 wasm` | ⏳ | - | nog uit te voeren |
| `bun --bun nuxt dev` opent UI | ⏳ | - | nog uit te voeren |

## Engine matrix — single thread

| Puzzle | baseline (ms) | opt-01 zobrist | opt-02 fwd-check | opt-03 mrv | opt-04 sym | opt-05 simd | opt-07 mitm | opt-08 bnb |
|---|---|---|---|---|---|---|---|---|
| level10 | — | — | — | — | — | — | — | — |
| level18 | — | — | — | — | — | — | — | — |
| level34 | — | — | — | — | — | — | — | — |
| level35 | — | — | — | — | — | — | — | — |
| level37 | — | — | — | — | — | — | — | — |
| evil-01 | — | — | — | — | — | — | — | — |

## Engine matrix — multi-worker (N=8)

| Puzzle | baseline | opt-06 work-steal | opt-final |
|---|---|---|---|
| level10 | — | — | — |
| level34 | — | — | — |
| level37 | — | — | — |
| evil-01 | — | — | — |

## Correctness — solutions-count per puzzle

(gevuld door SAT-oracle of door tegen baseline aan te leggen)

| Puzzle | baseline count | opt-01 | opt-02 | opt-03 | opt-04 | opt-05 | opt-07 | opt-08 | Z3 |
|---|---|---|---|---|---|---|---|---|---|
| level10 | — | — | — | — | — | — | — | — | — |
| level18 | — | — | — | — | — | — | — | — | — |
| level34 | — | — | — | — | — | — | — | — | — |
| level37 | — | — | — | — | — | — | — | — | — |

## Native AVX2 upper bound (opt-10)

Voor inzicht in hoeveel headroom de WASM-versie nog heeft. Niet productie-pad.

| Puzzle | wasm baseline | native AVX2 |
|---|---|---|
| level34 | — | — |
| level37 | — | — |

## Bekende issues

(geen — nog niets gemeten)

## Regressies

(geen — geen variants live)

## Browser-compat per variant

Vooral relevant voor opt-05 (SIMD-128 vereist Chrome 91+/FF 89+/Safari 16.4+) en opt-11 (WebGPU vereist Chrome 113+).

| Variant | Chrome | Firefox | Safari | Mobile |
|---|---|---|---|---|
| baseline | — | — | — | — |
| opt-05 simd | — | — | — | — |
| opt-11 webgpu | — | — | — | — |
