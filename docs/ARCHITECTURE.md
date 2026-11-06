# MATRIX architecture

**Audience:** coding agents and developers working on the MATRIX aerosol microphysics component.
**Describes:** the MAPL3 line, branch `release/MAPL-v3`.
**Verified against:** `MATRIX` @ `dfe783d` (2026-11-06).
**Placement:** as in the GEOSgcm fixture (`components.yaml` @ `f50622d`): key `MATRIX`, checked out at `src/Components/@GEOSgcm_GridComp/GEOSagcm_GridComp/GEOSphysics_GridComp/@GEOSchem_GridComp/@MATRIX`.
Other fixtures (for example GEOSadas, GEOSldas) may place this repository
elsewhere; paths inside this repository do not change.

**Build status on this line:** not built; not yet ported to MAPL3. On `release/MAPL-v3`, `@GEOSchem_GridComp/CMakeLists.txt` calls `esma_add_subdirectories(GOCART)` only, with the comment "Restore subdirs as components are ported to MAPL3."; `MATRIX` appears only in a commented-out list, as does `Chem_Shared` (from the Chemistry `Shared` directory, also in the commented-out list), its build dependency. The `esma_add_library` block that builds `GEOS_ChemGridComp.F90` (which adds this component) is commented out (`../CMakeLists.txt`); the file itself is present. The code below is the MAPL2 code, identical to the MAPL2 line, kept for a later port. Statements about the parent describe `GEOS_ChemGridComp.F90` as it is; the file is present but not compiled.

Parent: the Chemistry grid component, `@GEOSchem_GridComp`
(`../docs/ARCHITECTURE.md`; not repeated here).

## 1. Purpose

MATRIX ("Multiconfiguration Aerosol TRacker of mIXing state") is the GISS
modal aerosol microphysics code, wrapped as one ESMF gridded component named
`MATRIX` (`MATRIX_GridCompMod.F90`, module header). The repository is a small
GEOS interface layer (`MATRIX_GridCompMod.F90`, about 1,270 lines) over the
imported GISS kernel in `microphysics/` (`README`, "Importing/tracking code
from GISS/MATRIX"). The component advances 16 aerosol modes (mass and number
per mode, plus nitrate, ammonium and water) by one call to the kernel per grid
cell per step, and computes sea-salt and dust emissions itself.

## 2. Place in the hierarchy

- The Chemistry parent adds it with `MAPL_AddChild(GC, NAME='MATRIX', SS=MATRIX_SetServices)` when `ENABLE_MATRIX:` is true (`../GEOS_ChemGridComp.F90`, SetServices). Public entry: `SetServices` only (`MATRIX_GridCompMod.F90`, `public`).
- No children (`MATRIX_GridCompMod.F90`, SetServices: no `MAPL_AddChild`).
- Provides no `AERO` state and no aerosol-optics method: the module has no `AERO` reference and no `ESMF_MethodAdd` (`MATRIX_GridCompMod.F90`, grep). For the cross-repository `AERO` story see the parent's architecture document.
- Connections made by the parent for MATRIX (`../GEOS_ChemGridComp.F90`, SetServices): `AIRDENS` and `DELP` from `CHEMENV`; from `ACHEM` (only if both are enabled) `SO2`, `H2SO4`, `NH3`, `SOAG` (received as `SOA_GAS`), `pSO4_aq`, `pNH4_aq`. MATRIX is also a legal provider name in `GetProvider_`.

## 3. Entry points

All registered with `MAPL_GridCompSetEntryPoint`, one Run phase (`MATRIX_GridCompMod.F90`, SetServices).

| Routine | What it does |
|---|---|
| `SetServices` | Allocates the private state (`MATRIX_State`, stored under the name `MATRIX_state`); loads `MATRIX_GridComp.rc` into a private `ESMF_Config`; reads `verbose:` and `matrix:` (asserted equal to the compiled mechanism `MECH`); adds timer `RUN`; includes the generated import/internal/export spec headers; `MAPL_GenericSetServices`. |
| `Initialize_` | `MAPL_GenericInitialize`; `extract_`; stores grid, dimensions and time step; reads the resolution-dependent `f_emissions_seasalt:` and `f_emissions_dust:` rows with `Chem_UtilResVal`; calls the kernel setup routines (`SETUP_CONFIG`, species maps, `SETUP_DP0`, aerosol mass map, coagulation tensors and `SETUP_KIJ`, emission, `SETUP_KCI`, `SETUP_NPFMASS`); checks the mode indices are set. |
| `Run_` | Per step: (a) sea-salt emissions in two modes via `SeasaltEmission` scaled by `f_emiss_seasalt` over water points (`LWI < 1`); (b) dust emissions via `MAM_DustEmissionGOCART` and `MAM_DustEmission` (fine 0.1-2.5 um, coarse 2.0-10 um), scaled by `f_emiss_dust` and `GINOUX_DU`; (c) loops over every cell: copies the internal-state fields into the kernel vector `aerosol_`, fills `emissions_` (bottom layer only) from the imported emission fluxes and the computed sea-salt and dust emissions, calls `SPCMASSES` and `MATRIX`, copies results back, fills `TOTAL_*` exports and updates the `DGN_*` diameters. |
| `Finalize_` | `MAPL_GenericFinalize`, then `extract_`; no other work. |
| `extract_` | Helper: private state, grid, config, dimensions, time step. |

Run-loop details (`MATRIX_GridCompMod.F90`, Run_):
- Vertical index is reversed for the kernel (`MATRIX_L = lm - l + 1`); emissions and a bottom-layer loss factor `exp(-0.5d-2 * dt * g * airdens / delp)` apply only at `l == lm` (the bottom layer).
- Temperature `T`, relative humidity `min(RH2, 0.95)`, layer-mean pressure from `PLE`, and a fixed updraft speed `w_updraft_ = 0.05` are passed to the kernel.
- Before the kernel call, `aerosol_` and `gas_` values below `tiny` are reset to `1e3*tiny`; after the aerosol emissions are set, `aerosol_` is floored again at `tiny` (`MATRIX_GridCompMod.F90`, Run_).

## 4. State

Declared in `MATRIX_Registry.rc` (`ImportSpec`, `ExportSpec`, `InternalSpec`); the build generates `MATRIX_*Spec___.h` headers from it (`CMakeLists.txt`, `esma_generate_gocart_code`). Read the registry for field lists.
- Imports (49 registry rows): meteorology (`T`, `Q`, `RH2`, `PLE`, `ZLE`, `AIRDENS`, `DELP`, `U10M`, `V10M`, `USTAR`, `LWI`, `WET1`, `FRLAKE`, ...), the ACHEM gases, and emission fluxes `SO2_EMIS_*`, `BC_EMIS_*`, `OC_EMIS_*`, `SO4_EMIS_SHIP`, `GINOUX_DU` (`MATRIX_Registry.rc`, ImportSpec).
- Exports: `TOTAL_SU/DU/SS/OC/BC` (xyz) and `EMISS_SSA/SSC/SS/DUF/DUC/DU` (xy) (`MATRIX_Registry.rc`, ExportSpec).
- Internal state (`MATRIX_Registry.rc`, InternalSpec): per mode `N_<mode>` number, `M_<mode>_<species>` mass, `M_NH4`, `M_NO3`, `M_H2O`, and `DGN_<mode>` diameters. Modes: AKK, ACC, DD1, DS1, DD2, DS2, SSA, SSC, OCC, BC1, BC2, BC3, DBC, BOC, BCS, MXX. *Unverified:* which entries are `norestart` (columns not read in detail).
- Kernel working state (module variables of the `microphysics/` modules, for example `MATRIX_I/J/L`, `MATRIX_DIAMETER`, `MATRIX_VDDEP_AERO`) is allocated and freed within `Run_` or set in `Initialize_`.

## 5. Configuration that changes structure

- `ENABLE_MATRIX:` (parent `GEOS_ChemGridComp.rc`): adds the child.
- `MATRIX_GridComp.rc`: `verbose:`, `f_emissions_seasalt:`, `f_emissions_dust:` (six values, one per resolution class `a`-`f`, chosen by `Chem_UtilResVal`). `matrix:` is read (default 1) but is not in the file.
- Compile time: mechanism and mode count are fixed by preprocessor macros set in `CMakeLists.txt` (`target_compile_definitions`: `TRACERS_AMP TRACERS_AMP_M1 GEOS5_PORT`). `TRAMP_config.F90` defines `MECH`, `NMODES`, `NAEROVARS` per `TRACERS_AMP_M<n>` (M1: 16 modes). Changing mechanism needs a rebuild and changes the internal state.

## 6. Dependencies

- `Chem_Shared`, `MAPL`, `ESMF::ESMF` (`CMakeLists.txt`). Used from outside this repository: `Chem_UtilResVal`, `SeasaltEmissionMod`, `DustEmissionMod` (`MATRIX_GridCompMod.F90`, `use`); *Inferred:* they come from `Chem_Shared`.
- Kernel modules in `microphysics/` used by the interface: `AERO_CONFIG`, `AERO_PARAM`, `AERO_SETUP`, `AERO_COAG`, `AERO_NPF`, `AERO_DIAM`, `AMP_AEROSOL`, `CONSTANT`, and the external routines `MATRIX`, `SPCMASSES` (`MATRIX_GridCompMod.F90`, `use`, Run_).
- `microphysics/` map (GISS code, Fortran fixed form `.F` and `.F90`, imported from GISS ModelE MATRIX v2 032015; `README`):

| File | Role (from header comments) |
|---|---|
| `TRAMP_matrix.F` | `MATRIX`, top-level microphysics routine (gas/aerosol vectors in, tendencies applied). |
| `TRAMP_nomicrophysics.F` | `AERO_NOMICROPHYSICS`, the no-microphysics option (emissions and gas-particle partitioning only). |
| `TRAMP_drv.F` | module `AMP_AEROSOL`, driver variables for aerosol microphysics (GISS model-driven parts are under `#ifndef GEOS5_PORT`). |
| `TRAMP_config.F90`, `TRAMP_param.F` | `AERO_CONFIG` (mechanism 1-8 selection), `AERO_PARAM` (constants). |
| `TRAMP_setup.F`, `TRAMP_init.F` | `AERO_SETUP` (mode and species maps, setup routines). `TRAMP_init.F` is not in `CMakeLists.txt`. |
| `TRAMP_coag.F`, `TRAMP_npf.F`, `TRAMP_actv.F`, `TRAMP_diam.F`, `TRAMP_depv.F`, `TRAMP_dicrete.F`, `TRAMP_quad.F`, `TRAMP_subs.F`, `TRAMP_rad.F`, `CONST.F` | coagulation, new-particle formation, activation, diameters, deposition velocity, utilities, constants. `TRAMP_dicrete.F`, `TRAMP_rad.F` are not in `CMakeLists.txt`. |
| `TRAMP_isofwd2.F`, `TRAMP_isorev2.F`, `TRAMP_isocom2.F`, `isrpia.inc`, `TRAMP_thermo_isorr2.F` | ISORROPIA thermodynamic equilibrium code (University of Miami and others). |
| `TRAMP_eqsam_v03d.F90`, `TRAMP_thermo_eqsam.F` | EQSAM thermodynamics alternative; not in `CMakeLists.txt`. |
| `rundeck_opts.h` | GISS ModelE preprocessor flags; included only under `#ifndef GEOS5_PORT` in the built files (`TRAMP_config.F90`, `TRAMP_drv.F`), so not used in the GEOS build; `TRAMP_rad.F` includes it unconditionally but is not built. |

## 7. Known traps

- The kernel carries module-level state (`SAVE`d variables and `MATRIX_I/J/L`); the cell loop in `Run_` is serial and sets these indices per cell. *Inferred:* the kernel is not thread-safe.
- The vertical order is reversed for the kernel; the diameter array is indexed with the reversed level.
- Gas inputs `gas_(1:3)` and the in-cloud sulfate rate are set to `tiny`; the code that would use imported `H2SO4`, `NH3`, `SO2` is commented out (`MATRIX_GridCompMod.F90`, Run_). The imports exist but do not reach the kernel in this version.
- Emissions of sulfate, black and organic carbon enter only through the bottom layer.
- Mode index tables (`MATRIX_MASS_*`, `MATRIX_NUMB_*`) come from the compiled mechanism; copy-in and copy-out blocks in `Run_` list them by hand.

## 8. Decisions

See `decisions.md`.

## 9. Observations (noted, not fixed) and open questions

- `Run_` calls `MAPL_TimerOn/Off` for `-EMISSIONS` and `-MICROPHYSICS`, but `SetServices` adds only `RUN` outside `#ifdef __MATRIX_TODO__` (`MATRIX_GridCompMod.F90`). Question: is a timer that was never added silently ignored by MAPL?
- Copy-out in `Run_`: `N_SSA = aerosol_(MATRIX_NUMB_SSA)` has no `(i,j,l)` index, unlike the neighbouring statements.
- `Initialize_` calls `MATRIX_setup_dp0` twice; the final `_ASSERT(any(matrix_aerosol_indexes > 0), ...)` passes if any index is positive and carries the placeholder message `'needs informative message'`.
- `MATRIX_GridComp.rc` comment text and `MATRIX_Registry.rc` header say "GEOS-Chem Grid Component" (copied text).
- `TRAMP_drv.F` and `TRAMP_config.F90` include `rundeck_opts.h` only under `#ifndef GEOS5_PORT`, so that header is not used in the GEOS build; `TRAMP_drv.F` also keeps GISS ModelE `USE` statements and tables under the same guard.
- Question for maintainers: is MATRIX in current use? It has no `AERO` state or optics method, and the gas and aqueous-sulfate inputs are placeholders.
