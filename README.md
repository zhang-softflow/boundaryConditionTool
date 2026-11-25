# foamBC — search, generate and check OpenFOAM boundary conditions

English | [日本語](README.ja.md)

OpenFOAM (ESI, openfoam.com) ships about 250 finite-volume boundary conditions (fvPatchField) and about 40 point boundary conditions (pointPatchField, for mesh motion). Every time you set up a case you have to dig through headers and tutorials to find the type name, the mandatory entries, which types work for which field (U, p, k, ...), and the usual inlet/outlet/wall combinations.
**foamBC** parses the doc comments (`Description`, `Usage`, `\verbatim` examples, `\table` parameter tables) of every registered boundary condition, plus the `boundaryField` of about 3000 field files in the tutorials. With that database you can search, read the documentation, generate entries, **generate the entries of every field for one patch role (recipe)** and **check a case (check)**.

It is the sibling of [foamFO](../functionObjects) (function objects) and [foamFV](../fvOptions) (fvOptions).

| File | Content |
|---|---|
| `foamBC` | CLI (Python ≥ 3.8, standard library only) |
| `catalog.html` | Searchable offline catalog in a single file; open it in a browser |
| `bcDatabase.json` | Bundled database (built from OpenFOAM-v2606) |
| `catalog_template.html` | Template for the catalog |
| `skill/openfoam-boundaryconditions/` | Claude Code skill |
| `install.sh` | Installer: links the skill and sets up PATH |

## Install

```bash
./install.sh            # link the skill into ~/.claude/skills and print the PATH line
./install.sh --bashrc   # also append the PATH line to ~/.bashrc (makes a backup first)
./install.sh --uninstall
```

For an OpenFOAM version other than v2606, source that environment and build its database. This takes a few seconds:

```bash
source /usr/lib/openfoam/openfoam2412/etc/bashrc
foamBC build            # -> ~/.local/share/foamBC/bcDatabase-openfoam2412.json
```

The database is looked up in this order: `$FOAMBC_DB`, then `~/.local/share/foamBC/bcDatabase-<version>.json`, then the bundled `bcDatabase.json`. If the bundled database does not match your environment, foamBC prints a warning. Set `FOAMBC_QUIET=1` to silence it.

## Usage

### Check a case

```bash
foamBC check -case .
```
Matches the patches of `constant/polyMesh/boundary` (names and types) against the `boundaryField` of every field in `0/` (or `0.orig/`), prints a patch × field table and the problems found, and exits with status 1 on errors. It understands regular-expression keys, patch groups, `#includeEtc "caseDicts/setConstraintTypes"` and `#include`. It reports: missing patch entries, unknown type names (with a suggestion), types not available for the value type of the field, constraint-type mismatches (e.g. `zeroGradient` on an `empty` patch), wall functions on non-wall patches, required keywords missing according to the header table (warning) and entries matching no mesh patch (warning).

### Generate all fields for one patch role

```bash
foamBC recipe                                     # list the roles
foamBC recipe velocityInlet  -case . -patch inlet  --U '(10 0 0)' --I 0.05 --L 0.01
foamBC recipe pressureOutlet -case . -patch outlet --p 0 --insert
foamBC recipe wall           -case . -patch heater --Tw 350 --insert
foamBC recipe velocityInlet  --fields U,p,k,omega,nut --U 5      # without a case
```
Roles: `velocityInlet`, `flowRateInlet`, `pressureOutlet`, `totalPressureInlet`, `openAtmosphere`, `farfield`, `wall`, `movingWall`, `slipWall`, `symmetry`. The entry for each field in `0/` is chosen from the field name and dimensions (U, p, p_rgh, k, epsilon, omega, nut, nuTilda, alphat, T, alpha.*, other scalars/vectors). Cases with `p_rgh` and compressible cases (p in Pa) are detected; without `--p`/`--p0` the outlet/ambient pressure is 0 (incompressible) or the uniform internalField of `0/p` (compressible). Each entry is annotated with how often the tutorials use it for that role, and with the most used alternative when it differs. `$internalField` is avoided for fields with a `nonuniform` internalField. `--insert` refuses to write while `<placeholders>` remain.

### Single boundary conditions

| Command | Description |
|---|---|
| `foamBC search total pressure` | Full-text search over names, descriptions, parameter names and aliases (Japanese aliases work too) |
| `foamBC list [-g wallFunction] [-t inlet] [--type vector] [-u]` | List entries. Flags: T = used in tutorials, W = wall patches only, w = wall function, C = constraint type |
| `foamBC typical k [-r inlet]` | Most used types of a field per patch role in the tutorials |
| `foamBC show totalPressure --inherited` | Description, example, parameter table, value types, ancestry, tutorial usage |
| `foamBC example inletOutlet -f k -m 3` | Real entries from the tutorials |
| `foamBC add fixedValue -case . -patch inlet -field U [--insert]` | Print one patch entry; `--insert` replaces/appends it in `0/U` (backup `U.bak`) |
| `foamBC show solidBodyMotionDisplacement --point` | Point boundary conditions (`pointDisplacement`) |
| `foamBC build [--src DIR] [-o FILE]` / `foamBC html` | Build the database / regenerate `catalog.html` |

## Notes
- ESI OpenFOAM only. Foundation (openfoam.org) boundary conditions and keywords (e.g. `profile` of `flowRateInletVelocity`) are not covered. Finite-area and fvs patch fields are not covered.
- Patch roles in the statistics are guessed from the patch names of the tutorials.
- `check` is static: `$variables` and `#include` are expanded where possible; `#codeStream` and expressions are not evaluated. Boundary conditions from user libraries are reported as unknown.
- Tested: databases built from v2606 (source tree), v2512 and v2012; `check` on all 602 tutorial cases (the only remaining errors are type names that do not exist, e.g. `compressible::kqRWallFunction`); recipe + `--insert` run with v2512 simpleFoam/pitzDaily and interFoam/damBreak.

## License
GPL-3.0-or-later ([LICENSE](LICENSE)). The bundled `bcDatabase.json` and `catalog.html` contain text from the OpenFOAM doc comments (GPL-3.0-or-later, © OpenCFD Ltd. and others).

OPENFOAM® is a registered trade mark of OpenCFD Limited, producer and distributor of the OpenFOAM software via www.openfoam.com. This offering is not approved or endorsed by OpenCFD Limited.
