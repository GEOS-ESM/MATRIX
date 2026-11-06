# AGENTS.md - MATRIX

MATRIX: GEOS interface to the GISS MATRIX modal aerosol microphysics, one
ESMF gridded component (`MATRIX_GridCompMod.F90`, `SetServices`) with 16
aerosol modes. Nested repository of the Chemistry grid component.

**Describes:** the MAPL3 line, branch `release/MAPL-v3`.
**Verified against:** `MATRIX` @ `dfe783d` (2026-11-06).
**Placement:** GEOSgcm fixture (`components.yaml` @ `f50622d`), key `MATRIX`:
`src/Components/@GEOSgcm_GridComp/GEOSagcm_GridComp/GEOSphysics_GridComp/@GEOSchem_GridComp/@MATRIX`.
Parent: `../docs/ARCHITECTURE.md` (Chemistry; added when `ENABLE_MATRIX:` is true; on this line the parent's documents are not written yet; the MAPL2-line versions are on its `develop` branch).

**Build status on this line:** not built; not yet ported to MAPL3. On
`release/MAPL-v3`, `@GEOSchem_GridComp/CMakeLists.txt` calls
`esma_add_subdirectories(GOCART)` only, with the comment "Restore subdirs as
components are ported to MAPL3."; `MATRIX` appears only in a commented-out
list of subdirectories. Its build dependency `Chem_Shared` (from the Chemistry
`Shared` directory) is also in the commented-out list. The Chemistry top-level library
(`GEOS_ChemGridComp.F90`, which adds this component) is present but not
compiled: the `esma_add_library` block that builds it is commented out in
`../CMakeLists.txt`. The code in this repository is the MAPL2 code, identical
to the MAPL2 line, kept for a later port.

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
- Not built on this line (see build status above). The repository's own `CMakeLists.txt` has `esma_add_library` and depends on `Chem_Shared MAPL ESMF::ESMF`; it is not reached. It has no tests of its own.
