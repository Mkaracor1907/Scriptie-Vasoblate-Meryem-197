# EPA/PDT FEM model — pig 197 mesh pipeline

Pipeline that turns two separately segmented STL surfaces (artery + pancreas)
into a single conforming three-domain volume mesh (lumen, vessel wall, pancreas)
for simulating endovascular photo-activated ablation / photodynamic therapy
(EPA/PDT) around the splenic artery in COMSOL Multiphysics.

Builds on the model setup of Molenaar (VU Amsterdam / Amsterdam UMC).

---

## Pipeline

| Step | Script | Tool | Does |
|------|--------|------|------|
| 1 | `align_arteries_to_pancreas.m` | MATLAB | aligns artery to pancreas (translation) |
| 2 | `stap1_v17_ROI_meshlib.py` | Python / MeshLib | ROI crop + vessel wall + clearance |
| 3 | `stap2_tetgen_pig197_v8.m` | MATLAB / GIBBON + TetGen | conforming 3-domain volume mesh → NASTRAN |
| 4 | — | COMSOL | import via Mesh → Import → NASTRAN |

**Step 1.** Artery and pancreas come from different segmentations and do not
share a coordinate system. The artery is translated so its bounding-box centre
matches the pancreas. This corrects a *translation only* — it assumes both
segmentations share orientation and scale. Verify the overlap visually.

**Step 2.** Keeps only the treatment-relevant region. The pancreas is cropped to
the part contacting the splenic artery (contact zone found via lumen-to-pancreas
distance); the rest is discarded. The artery keeps the aortic stem + splenic
artery; other branches are removed. The wall is made by offsetting the lumen
(1.3 mm); a slightly larger offset is Boolean-subtracted from the pancreas so the
domains do not overlap. ROI bounds and clearance are set at the top of the script.

**Step 3.** The three surfaces are remeshed and passed *together* to TetGen with
one interior region point per domain, producing conforming nodes on the
interfaces (required for coupling physics across domains). Output: a NASTRAN file
with three domain IDs.

**Step 4.** Import via **Mesh → Import → NASTRAN**, not the geometry import.

---

## Files
- `01_arteries_reference_for_COMSOL.stl`, `02_pancreas_aligned_to_arteries_for_COMSOL.stl` — aligned inputs
- `pig197_3domains_Cropped_FIXED_BIGGER_cut_r2p0.nas` — final mesh (171,419 nodes, 1,017,909 elements)

## Requirements
MATLAB + GIBBON + TetGen · Python (meshlib, trimesh, numpy) · COMSOL 5.5

---

## Lessons learned (read before extending)

These cost time to discover — worth knowing if you build on this:

- **Do not mesh the segmented geometry inside COMSOL.** Merging surfaces with
  Form Union and re-meshing fails with "self-intersections not supported",
  inherent to image-segmented anatomy. Always generate the volume mesh externally
  (TetGen) and import it as a *mesh*, never as geometry.
- **Geometry import vs mesh import.** If you see a "Free Tetrahedral" error after
  importing, COMSOL is re-meshing — you imported into the geometry node instead of
  the mesh node.
- **Units.** Coordinates are in mm in the scripts, but the geometry imported into
  COMSOL already scales correctly to cm there (vessel ≈ 6–7 cm). Do not convert
  to mm inside COMSOL or everything becomes 10× too large.
- **Domain order.** TetGen may write the domain IDs in a different order than you
  expect. Verify which domain is the wall vs the lumen with a containment check
  (the lumen lies geometrically inside the wall). Assign physics accordingly.
- **Boolean failures give empty files.** If a Boolean subtract produces an empty
  STL (e.g. ~84 bytes), the balloon/offset likely pierces the vessel wall or the
  input is not watertight. Check watertightness before meshing.
- **Result looks "flat"/one colour?** That is usually a plot setting, not the
  physics: light fluence spans many orders of magnitude, so use a *logarithmic*
  colour scale and a volume plot instead of slices.

## Possible next steps
- Tighter, splenic-only ROI for a smaller, faster mesh.
- Add the diffuser/balloon as explicit domains (5-domain mesh) or as a source
  term in the light-distribution PDE (avoids meshing the thin diffuser).
