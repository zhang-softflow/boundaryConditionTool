# Changelog

## 1.0.0 — 2026-11-25
- First release.
- `foamBC` CLI: `build`, `list`, `search`, `show`, `example`, `typical`, `add` (`--insert`), `recipe` (10 patch roles, `--insert`), `check`, `html`.
- Detects `makePatchFields`, `makePatchFieldType`, `makePatchTypeField`, `addToPatchFieldRunTimeSelection`, `makePointPatchFields`, `makePointPatchTypeField` and named registrations (e.g. `overset` point fields); handles `.C`/`.cxx` sources and typedef'd classes.
- Wall-only detection from the sources (`isA<wallFvPatch>` + FatalError), constraint types, value types and ancestry.
- Tutorial statistics: usage per type, per field and per patch role.
- Per-version databases in `~/.local/share/foamBC`, bundled database for OpenFOAM-v2606.
- Offline `catalog.html` with ready-to-edit entries and a "typical per field" view.
- Claude Code skill `openfoam-boundaryconditions` and `install.sh`.
