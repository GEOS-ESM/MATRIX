# AGENTS.md - MATRIX

MATRIX: GEOS interface to the GISS MATRIX modal aerosol microphysics, one
ESMF gridded component (`MATRIX_GridCompMod.F90`, `SetServices`) with 16
aerosol modes. Nested repository of the Chemistry grid component.

**Describes:** branch `main`. **Verified against:** `MATRIX` @ `dfe783d` (2026-11-06).
**Placement:** GEOSgcm fixture (`components.yaml` @ `0f2e0fb`), key `MATRIX`:
`src/Components/@GEOSgcm_GridComp/GEOSagcm_GridComp/GEOSphysics_GridComp/@GEOSchem_GridComp/@MATRIX`.
Parent: `../docs/ARCHITECTURE.md` (Chemistry; added when `ENABLE_MATRIX:` is true).

## Documents
- `docs/ARCHITECTURE.md`: placement, entry points, state, configuration, `microphysics/` map, traps, observations.
- `docs/decisions.md`: previous architectural decisions.
- `README`: how the GISS code is imported. `CHANGELOG.md`: release notes.

## Where things are
- `MATRIX_GridCompMod.F90`: SetServices, Initialize_, Run_ (one phase), Finalize_, extract_.
- `MATRIX_Registry.rc`: import, export and internal specs (generate the `MATRIX_*Spec___.h` headers at build).
- `MATRIX_GridComp.rc`: `verbose:`, resolution-dependent emission factors. Installed to `etc`.
- `microphysics/`: GISS kernel (`TRAMP_*.F`), including ISORROPIA files. Imported code; keep edits minimal.

## Rules for agents
- Note bugs, do not fix them unless asked (see "Observations" in `docs/ARCHITECTURE.md`).
- Keep the documents current when code changes; cite `(file, routine)`, not line numbers.
- Do not read the large ISORROPIA files (`TRAMP_iso*.F`); headers suffice.
- MATRIX gives no `AERO` state or optics method; do not look for one here.
- Mechanism and mode count are compile-time (`CMakeLists.txt`, `TRACERS_AMP_M1`); changing them changes the registry.

## Build and test
- Built as part of GEOSgcm through the Chemistry parent (`CMakeLists.txt`: `esma_add_library`, depends on `Chem_Shared MAPL ESMF::ESMF`). It has no tests of its own.
