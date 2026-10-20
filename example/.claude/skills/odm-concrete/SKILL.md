---
name: odm-concrete
description: Build a 3D model of a concrete structure (bridge, pier, wall, tunnel) from drone photos with OpenDroneMap in Docker, using settings for low-texture concrete, then check point cloud quality with Open3D. Use when the user asks to process a site, run or rerun ODM, fix gaps in a model, or assess a point cloud of a structure.
---

# OpenDroneMap for concrete structures

Goal: a dense, clean 3D model of the structure and a short, honest quality report.
Work through the steps in order. Never modify, move or delete the original images.

## 1. Check the input

1. Find the site folder. ODM expects `<datasets>/<site>/images/`. Ask for the path if it is not clear.
2. Count the images and list the file types. ODM reads JPG and TIF; convert anything else first.
3. Check whether the photos carry GPS in EXIF (`exiftool -n -GPSLatitude <file>` or Python Pillow).
   - No GPS: the model has no real-world position or scale. Say so, and ask whether a `gcp_list.txt` or `geo.txt` is available.
4. Report back before running: image count, camera model, GPS yes or no.

## 2. Pick a profile

Concrete is low-texture: smooth, uniform grey surfaces give few matchable features, which shows up as images that fail to align and holes in flat faces. Start with **Standard** unless the user asks otherwise.

| Profile | Use when | ODM options |
| --- | --- | --- |
| Quick check | Confirming the images align before a long run | `--feature-quality medium --pc-quality low --skip-orthophoto --skip-report` |
| Standard concrete | First full run | `--feature-quality ultra --min-num-features 16000 --pc-quality high --mesh-octree-depth 12 --mesh-size 400000 --skip-orthophoto` |
| Low-texture rescue | Standard left images unaligned or large holes | Standard, plus `--min-num-features 32000 --matcher-neighbors 0` and, if noise is high, `--pc-filter 2.5` |

Notes:
- Keep the default `--sfm-algorithm incremental`. `planar` is for flat, straight-down mapping flights, not 3D structures.
- Drop `--skip-orthophoto` only if the user wants an orthophoto (for example of a deck seen from above).
- These are starting points. Change one thing at a time and say what changed.

## 3. Run ODM

Run without `-t` (Claude's shell has no terminal) and keep a log next to the project:

```bash
docker run --rm -v "<datasets>":/datasets opendronemap/odm \
  --project-path /datasets <site> <profile options> \
  2>&1 | tee "<datasets>/<site>/odm_run.log"
```

- Full runs can take hours. Run in the background when possible and check the log tail.
- To save time on a rerun, restart from the right stage with `--rerun-from <stage>`:
  - feature or matching options changed → `--rerun-from opensfm`
  - `--pc-*` options changed → `--rerun-from openmvs`
  - mesh options changed → `--rerun-from odm_meshing`
- If the run fails, quote the last error lines from the log and suggest one fix.

## 4. Check the outputs

Main files inside `<datasets>/<site>/`:

| Output | Path |
| --- | --- |
| Point cloud | `odm_georeferencing/odm_georeferenced_model.laz` |
| Filtered cloud (PLY) | `odm_filterpoints/point_cloud.ply` |
| Textured 3D model | `odm_texturing/odm_textured_model_geo.obj` |
| Orthophoto (if not skipped) | `odm_orthophoto/odm_orthophoto.tif` |
| ODM report (if not skipped) | `odm_report/report.pdf` |

## 5. Measure quality

Needs Python with `open3d` (and `laspy[lazrs]` only if there is no PLY). Install if missing:
`pip install open3d "laspy[lazrs]"`. Write this to a temporary file and run it with the site folder as argument:

```python
import json, sys
from pathlib import Path
import numpy as np
import open3d as o3d

IMG_EXT = {".jpg", ".jpeg", ".tif", ".tiff", ".png", ".dng"}

def load_cloud(project):
    for rel in ("odm_filterpoints/point_cloud.ply", "odm_georeferencing/odm_georeferenced_model.ply"):
        if (project / rel).exists():
            return o3d.io.read_point_cloud(str(project / rel)), rel
    laz = project / "odm_georeferencing/odm_georeferenced_model.laz"
    if laz.exists():
        import laspy
        las = laspy.read(str(laz))
        xyz = np.vstack([las.x, las.y, las.z]).T
        xyz = xyz - xyz.min(axis=0)  # avoid float precision loss on UTM coordinates
        return o3d.geometry.PointCloud(o3d.utility.Vector3dVector(xyz)), str(laz.relative_to(project))
    sys.exit("No point cloud found. Did the ODM run finish?")

def aligned_images(project):
    images = [f for f in (project / "images").glob("*") if f.suffix.lower() in IMG_EXT]
    rec = project / "opensfm/reconstruction.json"
    if not rec.exists():
        return None, len(images)
    shots = set()
    for r in json.loads(rec.read_text()):
        shots.update(r.get("shots", {}).keys())
    return len(shots), len(images)

project = Path(sys.argv[1] if len(sys.argv) > 1 else ".").resolve()
aligned, total = aligned_images(project)
pcd, src = load_cloud(project)
n = len(pcd.points)
sample = pcd.random_down_sample(500_000 / n) if n > 500_000 else pcd
nn = np.asarray(sample.compute_nearest_neighbor_distance())
_, kept = sample.remove_statistical_outlier(nb_neighbors=20, std_ratio=2.0)
ext = pcd.get_axis_aligned_bounding_box().get_extent()
if aligned is None:
    print(f"Images aligned:   unknown, {total} input images")
else:
    print(f"Images aligned:   {aligned} of {total} ({100 * aligned / max(total, 1):.0f}%)")
print(f"Point cloud:      {src}, {n:,} points")
print(f"Extent x, y, z:   {ext[0]:.2f} x {ext[1]:.2f} x {ext[2]:.2f} (metres if georeferenced)")
print(f"Point spacing:    median {np.median(nn):.4f}, 95th percentile {np.percentile(nn, 95):.4f}")
print(f"Likely outliers:  {100 * (1 - len(kept) / len(sample.points)):.1f}%")
```

Read the numbers this way (rules of thumb, not standards):
- **Images aligned** well below 100%: matching struggled. Go to the Low-texture rescue profile, or ask whether the flight had enough overlap on the faces with missing images.
- **Point spacing**: compare the median with the smallest defect the inspection must see (for example crack width or spalling size). If spacing is coarser, say the model cannot resolve that detail.
- **95th percentile spacing** much larger than the median: there are sparse areas or holes, usually on smooth or shadowed faces.
- **Likely outliers** of more than a few percent: noisy cloud; consider `--pc-filter 2.5` and `--rerun-from openmvs`.
- Without GPS the units are arbitrary, so report spacing only relative to the model.

## 6. Report to the user

Keep it short:
1. Profile and options used, and how long the run took.
2. The quality numbers from step 5.
3. What looks weak (for example "east pier face sparse").
4. One recommended next step: accept the model, rerun from a stage with a named change, or reshoot specific faces.
