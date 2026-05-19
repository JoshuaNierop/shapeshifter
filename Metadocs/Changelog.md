# Changelog — Shapeshifter

Format: `YYYY-MM-DD — [type] what — why`

Types: `add`, `change`, `fix`, `remove`, `rename`, `deprecate`.

---

## 2026-05-19

- **add** Fork JoshuaNierop/shapeshifter aangemaakt vanaf Anoesj/vormveranderaar — eigen werkruimte zonder upstream te raken
- **add** Werkbranch `josh/main` vanaf upstream `claude/puzzle-solver-multithreading-option-2` — baseline voor onze optimalisaties
- **add** Metadocs/ scaffold (Readme, Research, Plan, Todo, Planning, Testresults, Changelog) — single source of truth
- **add** `Metadocs/Research.md` met 13 onderzochte optimalisatie-richtingen + papers + referentie-repos
- **add** `Metadocs/Plan.md` met branch-per-variant aanpak (opt-00 t/m opt-10) en acceptance criteria
- **change** Rust toolchain 1.80.1 → 1.95.0 — wasm-pack 0.15 vereist edition2024
- **change** `.gitignore` schoongetrokken (`dist.nexus` op één regel splitsen, `bench-results/` toegevoegd)
