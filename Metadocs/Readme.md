---
name: Shapeshifter
category: Shelf
complexity: max
stack: [nuxt, vue, rust, wasm, bun, tailwind]
---

# Shapeshifter

Fork van Anoesj's `vormveranderaar` (Neopets Shapeshifter puzzle solver). Doel: alle realistische optimalisaties bouwen, benchmarken en vergelijken op één gedeelde puzzle-suite.

## Doel
- Origineel solver (TS in main): brute force met influence-bound pruning + dedup
- Anoesj's Claude-branches hebben Rust+WASM en depth-1 multi-worker slicing toegevoegd (factor ~12× single thread, ~N× via workers)
- Wij gaan **bovenop** dat ~10 verschillende optimalisatie-richtingen implementeren, elk in een eigen branch, en vergelijken op dezelfde bench-suite (level10/18/34/35/37 + nog te bouwen "hard" levels)

Geen merge naar Anoesj; alleen pushen naar `origin = JoshuaNierop/shapeshifter` (fork).

## Quickstart
```bash
# Setup
cd ~/Documents/Claude/Projects/Shelf/Shapeshifter
bun i

# Bouw wasm (alleen nodig na Rust-edits)
bun run build:wasm

# Dev server
bun --bun run dev

# Bench (één variant tegen JS)
bun --bun run wasm/solver/bench.mjs level34 7 both
```

## Key files
| File | Rol |
|---|---|
| `server/utils/shapeshifter/Puzzle.ts` | Originele TS solver (1066 regels) — referentie |
| `wasm/solver/src/lib.rs` | Rust solver (1999 regels op multithread-branch) — onze startbasis |
| `app/composables/useParallelSolver.ts` | JS-orchestrator voor depth-1 worker slicing |
| `app/utils/shapeshifterWorker.ts` | Web Worker entry (wasm init + dispatch) |
| `wasm/solver/bench.mjs` | Bench-harness — runs JS + wasm op puzzle-set |
| `Metadocs/Research.md` | Alle methoden/papers/repos waar we uit putten |
| `Metadocs/Plan.md` | Welke optimalisaties, in welke branch, met welk meetcriterium |
| `Metadocs/Testresults.md` | Bench-output per variant (gevuld na elke run) |

## Tech stack
- **Hosting:** Cloudflare Pages (overstap vanaf Netlify-static) of lokaal `bun run dev` voor benchen
- **Database:** geen
- **Auth:** geen
- **Frontend:** Nuxt 3 + Vue 3 + shadcn-nuxt + Tailwind
- **Solver kern:** Rust -> wasm32-unknown-unknown via wasm-bindgen + wasm-pack
- **Runtime:** Bun 1.2.21 (zowel server-side Nitro als bench-harness)

## Architectuur
Twee parallelle solver-paden:
1. **In-browser (default)** — Web Worker laadt wasm, roept `solve()` (single-thread) of N workers roepen `solve_slice(numWorkers, workerIndex, stopCb)` parallel
2. **Server-side** — `/api/calculate-solutions` POST -> Bun rekent met TS-solver (oude pad, niet aangeraakt op multithread-branch)

`SharedArrayBuffer` voor early-stop broadcasting vereist COOP/COEP headers (`app/public/_headers` + Nuxt `routeRules`).

## Deploy
Niet relevant in deze fase. We benchen lokaal; pas later UI live zetten.

## Status
- Fase: planning klaar; setup klaar; klaar om Fase 0 (bench-uitbreiding) te starten
- Blockers: geen — toolchain (Rust 1.95, wasm-pack 0.15, bun deps) operationeel
- Eigenaar: Joshua

## Links
- GitHub fork: https://github.com/JoshuaNierop/shapeshifter
- Upstream: https://github.com/Anoesj/vormveranderaar
- Live: n.v.t.
- Nexus: gesynct via `.nexus` dirty flag

## Git workflow
- `upstream` = Anoesj/vormveranderaar (read-only voor ons)
- `origin` = JoshuaNierop/shapeshifter (alle pushes hierheen)
- `josh/main` = baseline (= upstream `claude/puzzle-solver-multithreading-option-2`), waar alle feature-branches vanaf komen
- Elke optimalisatie krijgt een eigen branch `josh/opt-<naam>` zodat we ze tegelijk kunnen benchen
