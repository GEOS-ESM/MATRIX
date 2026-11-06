**Audience:** coding agents and developers working on MATRIX.
**Describes:** the MAPL2 line, branch `main`.
**Verified against:** `MATRIX` @ `dfe783d` (2026-11-06).
**Placement:** as in the GEOSgcm fixture (`components.yaml` @ `0f2e0fb`).
Other fixtures may place this repository elsewhere; paths inside it do not change.

# Previous architectural decisions

### Interface layer separate from the imported kernel
- Pattern: GEOS code lives in `MATRIX_GridCompMod.F90`; the GISS kernel is under `microphysics/` with renamed extensions and `GEOS5_PORT` guards (`README`; `CMakeLists.txt`, `GEOS5_PORT`).
- Likely reason: keep the vendor code close to upstream so new GISS versions can be re-imported. [Inferred]

### One compiled mechanism
- Pattern: mechanism 1 (16 modes) is selected at compile time by `TRACERS_AMP_M1`, and `SetServices` asserts the `matrix:` key equals it (`CMakeLists.txt`; `microphysics/TRAMP_config.F90`; `MATRIX_GridCompMod.F90`, SetServices).
- Likely reason: the internal-state field list in `MATRIX_Registry.rc` is fixed to the 16 modes. [Inferred]

### Emissions computed inside the component
- Pattern: sea-salt and dust emissions are computed in `Run_` from wind and surface imports with tuning factors from `MATRIX_GridComp.rc`; other species use imported fluxes (`MATRIX_GridCompMod.F90`, Run_).
- Likely reason: modal number and mass emissions per mode are needed, which the bulk emission providers do not supply. [Inferred]

### No `AERO` state or optics method
- Pattern: the component exposes only its own exports; no `AERO` state, no `ESMF_MethodAdd` (`MATRIX_GridCompMod.F90`, grep).
- Question for maintainers: is the radiation coupling for MATRIX aerosols intended to be added later?
