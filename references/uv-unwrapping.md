# UV Unwrapping: Production Reference

## Philosophy
UVs are the foundation of every texture. Bad UVs = broken normals, stretched textures, wasted texel density, failed bakes. A senior artist treats UV unwrapping as seriously as modeling. There are no shortcuts in production.

---

## The Two UV Channel Standard (AAA/Game Pipeline)

| Channel | Purpose | Rules |
|---------|---------|-------|
| **UV0** (Channel 0) | Texturing — diffuse, normal, ORM | Can overlap (tiling, mirroring OK) |
| **UV1** (Channel 1) | Lightmap baking (UE5/Unity) | **Must be non-overlapping, 0–1 space, margins 4–8px** |

Never merge these. Always deliver both for hero assets.

---

## Seam Strategy by Asset Type

### Hard Surface (Architecture, Props, Vehicles)
- Place seams on **hard edges** (edges with sharp normals)
- Seams on **interior/hidden faces** first
- Use **angle-based unwrap** for complex shapes
- Keep **top/front/side faces** as continuous islands when possible

### Organic (Rocks, Debris, Nature)
- Use **Smart UV Project** as a starting point, then manually clean up
- Minimize seams visible at primary camera angle
- Rocks: seams on bottom/back faces
- Foliage cards: flat planes, no seams needed (just scale/position islands)

### Modular Kit Pieces (KitBash style)
- **Tile-friendly**: UVs should allow tiling textures without obvious seams at junction points
- Keep walls/floors with UVs that respect texel density across the entire kit
- Trim sheets: UVs must map to specific rows/columns of the trim atlas

---

## Texel Density Standard

Texel density = pixels per meter. Consistency across a scene is critical.

| Asset Tier | Texel Density | Texture Res | Use Case |
|-----------|--------------|-------------|---------|
| Hero Prop | 512–1024 px/m | 2048–4096 | Signature pieces, close-up |
| Mid Prop | 256–512 px/m | 1024–2048 | Standard scene dressing |
| Filler/BG | 64–256 px/m | 512–1024 | Background, crowd objects |
| Terrain | 16–64 px/m | 4096 tiling | Large ground surfaces |

**Rule**: Match texel density across objects that will be next to each other. A hero prop next to a wall with 10× the texel density will look inconsistent.

---

## Production UV Unwrap Script

```python
import bpy

def smart_unwrap_selected(method='ANGLE_BASED', margin=0.005, correct_aspect=True):
    """
    Production unwrap on selected objects.
    Handles seam marking from sharp edges and unwraps UV0.
    
    method: 'ANGLE_BASED' (organic) | 'CONFORMAL' (hard surface)
    margin: island margin in UV space (0.005 = 0.5%)
    """
    selected_objs = [obj for obj in bpy.context.selected_objects if obj.type == 'MESH']
    
    if not selected_objs:
        print("[UV] No mesh objects selected.")
        return

    for obj in selected_objs:
        bpy.context.view_layer.objects.active = obj
        bpy.ops.object.mode_set(mode='EDIT')
        bpy.ops.mesh.select_all(action='SELECT')

        # Mark seams from sharp edges (production standard for hard surface)
        bpy.ops.mesh.mark_seam(clear=True)  # Clear existing seams
        bpy.ops.mesh.edges_select_sharp(sharpness=0.523599)  # ~30 degrees threshold
        bpy.ops.mesh.mark_seam(clear=False)

        # Unwrap
        bpy.ops.uv.unwrap(
            method=method,
            margin=margin,
            correct_aspect=correct_aspect,
            use_subsurf_data=False
        )

        bpy.ops.object.mode_set(mode='OBJECT')
        print(f"[UV] Unwrapped: {obj.name} | Method: {method}")

    bpy.ops.object.select_all(action='DESELECT')
    print(f"[UV] Done. {len(selected_objs)} objects processed.")

smart_unwrap_selected()
```

---

## Lightmap UV Generator (UV Channel 1)

```python
import bpy

def generate_lightmap_uvs(objects=None, margin=0.01, resolution_hint=512):
    """
    Add or replace UV Channel 1 with lightmap-ready UVs.
    Non-overlapping, fills 0-1 space efficiently.
    margin: pixel margin normalized (0.01 ≈ 5px at 512res)
    """
    targets = objects or [obj for obj in bpy.context.selected_objects if obj.type == 'MESH']
    
    for obj in targets:
        bpy.context.view_layer.objects.active = obj
        mesh = obj.data

        # Ensure UV0 exists
        if len(mesh.uv_layers) == 0:
            mesh.uv_layers.new(name="UV0_Texture")
            print(f"[UV] Created UV0 on {obj.name}")

        # Add UV1 if not present, or get existing
        if len(mesh.uv_layers) < 2:
            lm_layer = mesh.uv_layers.new(name="UV1_Lightmap")
        else:
            lm_layer = mesh.uv_layers[1]
            lm_layer.name = "UV1_Lightmap"

        # Set as active for editing
        mesh.uv_layers.active = lm_layer

        bpy.ops.object.mode_set(mode='EDIT')
        bpy.ops.mesh.select_all(action='SELECT')
        bpy.ops.uv.lightmap_pack(
            PREF_CONTEXT='ALL_FACES',
            PREF_PACK_IN_ONE=True,
            PREF_NEW_UVLAYER=False,
            PREF_BOX_DIV=12,
            PREF_MARGIN_DIV=margin
        )
        bpy.ops.object.mode_set(mode='OBJECT')

        # Reset active UV to UV0 for texturing
        mesh.uv_layers.active = mesh.uv_layers[0]
        print(f"[UV] Lightmap UV1 generated on: {obj.name}")

    print(f"[UV] Lightmap UVs done for {len(targets)} objects.")

generate_lightmap_uvs()
```

---

## Texel Density Checker & Normalizer

```python
import bpy
import bmesh
import math

def get_texel_density(obj, texture_width=2048, texture_height=2048):
    """
    Calculate average texel density (px/m²) for a mesh object.
    Returns density value and a rating.
    """
    if obj.type != 'MESH':
        return None

    bm = bmesh.new()
    bm.from_mesh(obj.data)
    bm.transform(obj.matrix_world)

    uv_layer = bm.loops.layers.uv.active
    if not uv_layer:
        bm.free()
        return None

    total_uv_area = 0.0
    total_world_area = 0.0

    for face in bm.faces:
        # World space area
        world_area = face.calc_area()
        total_world_area += world_area

        # UV space area
        uvs = [loop[uv_layer].uv for loop in face.loops]
        uv_area = 0.0
        n = len(uvs)
        for i in range(n):
            j = (i + 1) % n
            uv_area += uvs[i].x * uvs[j].y
            uv_area -= uvs[j].x * uvs[i].y
        uv_area = abs(uv_area) * 0.5
        total_uv_area += uv_area

    bm.free()

    if total_world_area == 0:
        return 0

    # Texel density in px/m
    texel_density = math.sqrt((total_uv_area * texture_width * texture_height) / total_world_area)
    
    # Rating
    if texel_density >= 512:
        rating = "Hero ✅"
    elif texel_density >= 256:
        rating = "Mid ✅"
    elif texel_density >= 64:
        rating = "BG/Filler ⚠️"
    else:
        rating = "Too Low ❌"

    print(f"[UV] {obj.name}: {texel_density:.0f} px/m → {rating}")
    return texel_density

# Run on all selected meshes
for obj in bpy.context.selected_objects:
    if obj.type == 'MESH':
        get_texel_density(obj)
```

---

## Common UV Anti-Patterns

| Problem | Symptom | Fix |
|---------|---------|-----|
| **Stretching** | Texture looks warped/elongated | Re-unwrap with correct aspect, check seam placement |
| **Overlapping islands (UV1)** | Lightmap bake bleeds | Re-generate UV1 with lightmap_pack |
| **Islands outside 0-1 space** | Missing textures in engine | Select all → Pack Islands |
| **Inconsistent texel density** | Objects look different resolution | Normalize with texel density tools |
| **No UV1 for baked assets** | UE5 lightmap errors | Always add UV1 for static meshes |
| **Mirrored UVs on UV1** | Bake artifacts | Only UV0 can use mirroring |

---

## Quick Reference: UV Operators (bpy)

```python
# Unwrap (edit mode, faces selected)
bpy.ops.uv.unwrap(method='ANGLE_BASED', margin=0.005)

# Smart UV Project
bpy.ops.uv.smart_project(angle_limit=66, island_margin=0.01)

# Pack islands
bpy.ops.uv.pack_islands(margin=0.005)

# Average islands scale (texel density normalization)
bpy.ops.uv.average_islands_scale()

# Minimize stretch
bpy.ops.uv.minimize_stretch(iterations=500)

# Seams from islands (reverse: island boundaries → seams)
bpy.ops.uv.seams_from_islands()
```