# Environment Modeling & Asset Pipeline Reference

## AAA Environment Art Philosophy
Environment art for KitBash/Fab is about **modularity, reusability, and art direction clarity**. Every asset must work in isolation AND as part of a kit. Think in kits, not individual props.

---

## Modular Kit Design Principles

### The Grid System
All kit pieces must snap to a grid. Typical scales:
- **Small props**: 0.25m, 0.5m, 1m grid
- **Architectural modules**: 1m, 2m, 4m grid  
- **Hero assets**: No grid constraint, but origin must be logical

### Kit Categories (KitBash standard)
```
Kit/
├── Foundation/     # Floors, bases, platforms
├── Walls/          # Straight, corner, door, window variants
├── Roof/           # Flat, pitched, edge caps
├── Props_Hero/     # High-detail standalone pieces
├── Props_Filler/   # Small repeatable details
├── Trim/           # Edge decoration, borders
└── Decals/         # Damage, wear, markings (planes with alpha)
```

---

## Scene Organization Script

```python
import bpy

def setup_production_scene(scene_name="ENV_Untitled"):
    """
    Initialize a production-ready scene with proper collection hierarchy.
    Run this at the start of every new environment project.
    """
    scene = bpy.context.scene
    scene.name = scene_name

    # Unit settings: real-world scale
    scene.unit_settings.system = 'METRIC'
    scene.unit_settings.length_unit = 'METERS'
    scene.unit_settings.scale_length = 1.0

    # Collection hierarchy
    collection_structure = {
        f"ENV_{scene_name}": [
            "GEO_Terrain",
            "GEO_Architecture",
            "GEO_Props_Hero",
            "GEO_Props_Filler",
            "GEO_Foliage",
            "GEO_Decals",
            "LGT_Setup",
            "CAM_Setup",
            "REF_Blockout"  # Placeholder geometry, hidden at export
        ]
    }

    root_col = bpy.data.collections.new(f"ENV_{scene_name}")
    bpy.context.scene.collection.children.link(root_col)

    children = collection_structure[f"ENV_{scene_name}"]
    for child_name in children:
        child_col = bpy.data.collections.new(child_name)
        root_col.children.link(child_col)

    # REF layer: exclude from render, viewable in viewport
    layer_collection = bpy.context.view_layer.layer_collection
    def find_layer_col(layer_col, name):
        if layer_col.name == name:
            return layer_col
        for child in layer_col.children:
            result = find_layer_col(child, name)
            if result:
                return result
        return None

    ref_layer = find_layer_col(layer_collection, "REF_Blockout")
    if ref_layer:
        ref_layer.exclude = False  # Visible in viewport
        ref_layer.holdout = False

    print(f"[PIPELINE] Scene '{scene_name}' initialized with AAA collection hierarchy.")

setup_production_scene()
```

---

## Asset Naming & Rename Batch Script

```python
import bpy

def batch_rename_assets(prefix="SM", category="Prop"):
    """
    Rename selected objects to AAA naming convention.
    Select objects before running.
    Usage: batch_rename_assets(prefix="SM", category="Rock")
    → SM_Rock_A, SM_Rock_B, SM_Rock_C...
    """
    selected = bpy.context.selected_objects
    letters = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
    
    for i, obj in enumerate(selected):
        variant = letters[i % len(letters)]
        number = (i // len(letters))
        suffix = f"_{number:02d}" if number > 0 else ""
        obj.name = f"{prefix}_{category}_{variant}{suffix}"
        # Also rename mesh/light/camera data block
        if obj.data:
            obj.data.name = obj.name

    print(f"[PIPELINE] Renamed {len(selected)} objects with prefix {prefix}_{category}")

# Example usage:
# batch_rename_assets(prefix="SM", category="UrbanWall")
```

---

## LOD Generation Strategy

### Manual LOD Workflow (Blender native)
1. Duplicate LOD0 → rename `_LOD1`
2. Apply **Decimate modifier** at 0.5 ratio
3. Review silhouette — manually fix if broken
4. Repeat for LOD2 (0.25) and LOD3 (0.1)
5. Export all LODs in single FBX → UE5 auto-detects `_LOD` suffix

### Target Triangle Counts
| Asset Type | LOD0 | LOD1 | LOD2 | LOD3 |
|-----------|------|------|------|------|
| Hero Prop (signature) | 10k–30k | 8k | 3k | 800 |
| Mid Prop | 2k–8k | 2k | 800 | 200 |
| Filler/Dressing | 200–1k | 500 | 150 | 50 |
| Modular Wall Piece | 500–2k | 800 | 300 | 100 |

---

## FBX Export for UE5 (Production Settings)

```python
import bpy

def export_for_ue5(filepath, selected_only=True):
    """
    Export with settings validated for UE5 import.
    Apply transforms before calling this!
    """
    bpy.ops.export_scene.fbx(
        filepath=filepath,
        use_selection=selected_only,
        apply_unit_scale=True,
        apply_scale_options='FBX_SCALE_ALL',
        axis_forward='-Z',
        axis_up='Y',
        object_types={'MESH', 'ARMATURE'},
        use_mesh_modifiers=True,
        mesh_smooth_type='FACE',
        use_mesh_edges=False,
        use_tspace=True,
        add_leaf_bones=False,
        primary_bone_axis='Y',
        secondary_bone_axis='X',
        bake_anim=False,
        path_mode='COPY'
    )
    print(f"[EXPORT] FBX exported to: {filepath}")

# CRITICAL: Always apply transforms before export
# bpy.ops.object.transform_apply(location=False, rotation=True, scale=True)
```

---

## Common Topology Rules

- **No n-gons** on curved or deforming surfaces (quads only)
- **N-gons OK** on flat, non-deforming faces (floor panels, walls)
- **Edge loops** must support silhouette at LOD0 distance
- **Bevels** on hard edges: 1–2 segment bevel, avoid support loops spam
- **Backface culling** enabled: no interior-only geometry
- **Manifold mesh**: no open edges on solid props (watertight)