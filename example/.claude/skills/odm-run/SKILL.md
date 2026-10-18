---
name: odm-run
description: Process a bridge site's drone images into a 3D model with OpenDroneMap in Docker. Use when the user asks to run ODM, process a site, or build a model or point cloud.
---

# Run OpenDroneMap on a site

1. Confirm the site folder exists under data/<site>/images.
2. Run ODM in Docker, mounting the data folder:
   docker run -ti --rm -v "$PWD/data":/datasets opendronemap/odm --project-path /datasets <site>
3. Check that outputs were written, then report what was produced.
4. Never modify or delete the original images.
