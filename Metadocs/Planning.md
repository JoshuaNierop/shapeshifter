# Planning — Shapeshifter

Gefaseerde roadmap. Afgeronde milestones migreren naar `Changelog.md`. Volledige technische plan staat in `Plan.md`, theoretische onderbouwing in `Research.md`.

## Milestone 1 — Setup + meetbasis (target: 2026-05-20)

Doel: we kunnen elke variant betrouwbaar meten en automatisch vergelijken.

- [x] P0 — Fork + remotes config
- [x] P0 — Rust toolchain
- [~] P0 — wasm-pack
- [ ] P0 — josh/main bouwt + runt (UI in browser, bench.mjs werkt)
- [ ] P0 — Bench-uitbreiding (`josh/opt-00-bench`) — volledige suite, JSON-output, statistics
- [ ] P0 — Baseline meting van josh/main op alle puzzles

## Milestone 2 — Eerste optimalisatie-ronde (target: 2026-05-23)

Implementeer en bench de 6 P1-varianten parallel. Elk in eigen branch, elk individueel gemeten.

- [ ] P1 — opt-01 transposition table met Zobrist
- [ ] P1 — opt-02 per-cell forward checking
- [ ] P1 — opt-03 MRV piece ordering
- [ ] P1 — opt-04 symmetry breaking
- [ ] P1 — opt-05 SIMD-128 v128 intrinsics
- [ ] P1 — opt-06 dynamic work-stealing

Acceptance: alle 6 branches mergeable in josh/main zonder conflicts (geen overlap in files), Testresults.md gevuld.

## Milestone 3 — Tweede ronde + combineren (target: 2026-05-26)

- [ ] P2 — opt-07 Meet-in-the-Middle
- [ ] P2 — opt-08 branch-and-bound
- [ ] P2 — opt-09 SAT-oracle (Z3) voor cross-verify
- [ ] P3 — opt-10 native AVX2 upper-bound
- [ ] P1 — opt-final: combineer top-3 winners

Acceptance: een variant verslaat josh/main met factor ≥3× op een evil-level.

## Milestone 4 — Documentatie + presentatie (target: 2026-05-28)

- [ ] P1 — `Testresults.md` matrix op één scrollscherm
- [ ] P1 — `Architecture.md` met diagram van solver-paden
- [ ] P1 — Decision-log: welke varianten produktie-pad worden, welke niet
- [ ] P2 — README-update voor mensen die de fork bekijken
- [ ] P2 — Korte schriftelijke samenvatting voor Anoesj (mag ie zien als-ie wil)

## Milestone 5 — Optioneel (geen target)

- [ ] P3 — WebGPU compute exploratie
- [ ] P3 — wasm-bindgen-rayon true threading
- [ ] P3 — UI-improvements: live progress, multi-variant compare in-browser
- [ ] P3 — Deploy naar Cloudflare Pages onder Joshua's domain

## Afgewezen / niet doen

- **Dancing Links / Algorithm X.** Past niet op de mod-N puzzle-structuur (zou tile-overlap forbidden vereisen, wat hier wel mag).
- **Volledige SAT-pad als productie-solver.** Z3 op grote puzzels is wisselvallig; wel als oracle.
- **Pattern databases.** State space te groot om te precomputen.
- **Merge naar Anoesj/vormveranderaar.** We werken in eigen fork. Anoesj heeft eigen Claude-sessies, dat is van hem.

## Werkwijze beslismomenten

Voor we Milestone 2 start: Joshua tekent af op `Plan.md`. Aanpassingen na dat moment = nieuwe Todo-item, niet sluipende wijziging.

Voor combineer-fase (M3 eind): we kijken naar de meetmatrix en kiezen welke winners samen produktie-pad worden.
