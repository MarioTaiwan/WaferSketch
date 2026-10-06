# WaferSketch 1.3

WaferSketch is a mask-to-wafer fabrication design validator built around one shared **material voxel state**. Lithography is assumed to transfer a DXF/GDS mask faithfully into photoresist. Etching, deposition, masking, lift-off and inspection all operate on the same material state.

## Main changes in 1.3

### 1. Adaptive Z sampling

The simulation no longer needs one uniform Z voxel thickness through the full wafer.

The **Voxel simulation** panel now contains:

```text
XY voxel size
Active-surface Z size
Untouched bulk Z size
Mask-region margin
Maximum voxels
```

For example, a 500 µm Si wafer can begin with:

```text
active-surface Z = 1 µm
untouched bulk Z = 20 µm
```

The top/bottom process regions and film interfaces remain finely sampled, while the middle of untouched Si is stored as thick slabs. Before a plasma or wet-etch front reaches a coarse slab, WaferSketch subdivides that slab into fine Z layers and then runs the cellular process on those children.

This is intentionally a **surface-adaptive layered grid**, not a full octree. X/Y resolution remains user controlled. The approach avoids most of the wasted cells caused by a uniform fine Z grid while retaining simple, robust neighbour relationships for the crystal cellular automaton.

Adaptive refinement obeys the `Maximum voxels` setting as a soft memory cap. If the requested local Z resolution cannot fit, WaferSketch relaxes the local refinement and reports a warning.

### 2. Thin-film-driven refinement

Surface-adding processes can request Z refinement finer than the general active-surface setting. For example, with a 1 µm active Z size and 50 nm evaporation, the surface layer is subdivided toward 50 nm before metal is added, if the memory cap permits it.

This applies to:

- Si3N4 formation
- SiO2 formation / deposition
- photoresist
- metal evaporation

Partial fill is still retained as a fallback when an exact thin-film subdivision would exceed the memory cap.

### 3. Isotropic BOE etch

A new **Isotropic BOE etch** process is available. It is an immersion process with material-specific rates:

```text
Duration
SiO2 rate
Si3N4 rate
Si rate
Resist rate
Metal rate
```

The default intent is BOE-like behaviour: SiO2 etches quickly while Si, metal and resist can be zero-rate blockers.

The solver first identifies empty voxels connected to the external liquid, then propagates an isotropic 26-neighbour travel-time front through materials with nonzero rate. This allows lateral undercut beneath a mask. A zero-rate material cannot be crossed.

### 4. SiO2 material

WaferSketch now contains an explicit `SiO2` material and an **Oxide formation / deposition** process so sacrificial-oxide / BOE release sequences can be represented.

## Crystalline Si wet etching

WaferSketch retains the 1.2 crystal-aware continuous cellular automaton.

Each exposed Si voxel is treated as a small crystal element. Its local digital surface normal is transformed into the wafer crystal axes and its removal rate is derived from:

```text
R100
R110
R111
```

The wafer properties contain:

```text
Surface orientation:       (100) / (110) / (111)
In-plane crystal rotation: degrees
```

Slow crystallographic configurations remain longer and shield Si behind them, producing faceted staircase surfaces. The anisotropic wet etch is an immersion process; there is no top/bottom etchant-source selector.

For a normal mapped by cubic symmetry to

```text
Nz >= Nx >= Ny >= 0
```

the base interpolation is

```text
R = [R100 (Nz-Nx) + R110 (Nx-Ny) + R111 Ny] / Nz
```

with a small principal-facet capture tolerance for coarse digital `{100}`, `{110}` and `{111}` staircases.

## Process model

### Material state

```text
material[z,y,x] = EMPTY / Si / Si3N4 / SiO2 / Resist / Metal
fill[z,y,x]     = 0..255 local occupied fraction
support[z,y,x]  = support provenance for deposited metal
z_edges         = nonuniform physical Z cell boundaries
```

### Lithography

DXF/GDS polygons are transferred directly into photoresist. Optical exposure is not simulated.

### Directional plasma etch

Each material has an independent etch rate. A zero-rate material blocks propagation. If a material is removed, remaining process time can continue into newly exposed material beneath it. Coarse Z slabs in the reachable band are refined before etching.

### Crystalline anisotropic Si etch

- immersion process;
- liquid reaches connected empty space from both wafer-normal exterior volumes;
- Si3N4, SiO2, resist and metal block access unless separately removed;
- local Si removal depends on crystal orientation;
- coarse Z slabs are refined before the active crystal front reaches them.

### Isotropic BOE etch

- immersion process;
- 26-neighbour isotropic propagation using physical X/Y/Z distances;
- material-specific rates;
- supports lateral undercut under resist/metal/nitride masks;
- zero-rate materials remain true barriers.

### Metal evaporation

Ballistic first-hit deposition from the selected evaporation side. The first physical material reached by each source ray receives metal. The surface Z stack is adaptively refined to represent thin deposited films when possible.

### Resist stripping / lift-off

Resist is removed together with metal whose support provenance is resist. Metal deposited through an opening onto Si, Si3N4 or SiO2 remains.

## Inspection

The inspection views consume the adaptive voxel state directly:

- Top / bottom projection
- X-Z section
- Y-Z section
- local 3D voxel context with full-wafer wireframe outline
- 2D zoom controls
- CAD-style snapping and A/B measurement
- Z-only cartoon scale

X-Z/Y-Z rendering uses the true nonuniform Z edges, so thick bulk slabs and refined surface layers are displayed at their physical thicknesses in normal mode. Cartoon mode changes only display Z; X/Y and all measurement coordinates remain physical.

## Performance and memory

Adaptive Z is designed to avoid spending fine cells on inactive bulk. It does **not** yet adapt X/Y. For a very large lateral mask region, XY sampling can still dominate memory.

Refinement can increase the number of Z layers during a recipe. `Maximum voxels` is checked during subdivision. If a requested thin-film or etch-front resolution cannot fit, WaferSketch keeps a coarser local layer and reports the effective limitation.

The crystalline Si cellular-automaton inner loop is compiled with **Numba**. The first crystalline run in a new environment may pause while Numba compiles the kernel.

## Windows 10 installation

If you already have a WaferSketch virtual environment:

```bat
cd WaferSketch_1.3
<path-to-existing-venv>\Scripts\activate
python -m pip install -r requirements.txt
python run.py
```

For a new installation:

```bat
cd WaferSketch_1.3
INSTALL_WINDOWS.bat
```

or manually:

```bat
py -3.12 -m venv .venv
.venv\Scripts\activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python run.py
```

## Dependencies

- Python 3.12
- NumPy
- SciPy
- Numba
- Shapely
- ezdxf
- gdstk
- PySide6
- PyVista / pyvistaqt

## Regression tests

```bat
set PYTHONPATH=.
python tests\test_voxel_engine.py
python tests\test_display_transform.py
```

The regression suite now covers:

- Si3N4 blocking crystalline wet etching;
- opening nitride before Si etching;
- anisotropic depth scaling with rate and time;
- local through-etch without deleting remote Si;
- square `(100)` openings narrowing with depth;
- wafer orientation / rotation recipe round-trip;
- deposition through resist openings and lift-off provenance;
- metal blocking wet etching;
- coarse untouched bulk Z layers;
- automatic Z subdivision around an active anisotropic front;
- thin metal deposition triggering finer surface Z sampling;
- isotropic BOE lateral undercut beneath resist;
- cartoon Z scaling.

## Important limitations

WaferSketch remains a design-validation simulator, not full process TCAD.

- Adaptive refinement is currently **Z-only**; X/Y remain uniform.
- Crystal facets are staircase approximations whose lateral accuracy depends on XY voxel size.
- BOE isotropy is a 26-neighbour voxel approximation, not a continuum level-set solver.
- `R100`, `R110`, and `R111` are the only crystal-rate families currently used.
- Thin-film subdivision may be relaxed when the voxel-memory cap would otherwise be exceeded.
- Wet etchant enters from the top/bottom exterior padding of the local simulation crop, not from the distant perimeter of an entire physical wafer.
- Evaporation is ballistic first-hit transport without scattering or re-emission.
