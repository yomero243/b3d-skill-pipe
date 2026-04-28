# Blender Python (bpy) Production Patterns

## Safety First: Always Context-Check

```python
import bpy

# Never assume active object exists
obj = bpy.context.active_object
if not obj:
    print("[ERROR] No active object. Select something first.")
    raise RuntimeError("No active object selected.")

# Never assume edit mode
if bpy.context.mode != 'OBJECT':
    bpy.ops.object.mode_set(mode='OBJECT')
```

---

## Pattern: Safe Object Mover to Collection

```python
def move_to_collection(obj, collection_name):
    """Move object to named collection, creating it if needed."""
    col = bpy.data.collections.get(collection_name)
    if not col:
        col = bpy.data.collections.new(collection_name)
        bpy.context.scene.collection.children.link(col)
    for c in list(obj.users_collection):
        c.objects.unlink(obj)
    col.objects.link(obj)
```

## Pattern: Apply All Transforms (critical before export)

```python
def apply_transforms(objects=None, scale=True, rotation=True, location=False):
    """Apply transforms on a list of objects or all selected."""
    targets = objects or bpy.context.selected_objects
    bpy.ops.object.select_all(action='DESELECT')
    for obj in targets:
        obj.select_set(True)
        bpy.context.view_layer.objects.active = obj
    bpy.ops.object.transform_apply(
        location=location,
        rotation=rotation,
        scale=scale
    )
    bpy.ops.object.select_all(action='DESELECT')
```

## Pattern: PBR Material Setup

```python
def create_pbr_material(name, base_color=(0.5,0.5,0.5,1), metallic=0.0, roughness=0.5):
    """Create a production PBR material with Principled BSDF."""
    mat = bpy.data.materials.new(name=f"MI_{name}")
    mat.use_nodes = True
    nodes = mat.node_tree.nodes
    nodes.clear()

    output = nodes.new('ShaderNodeOutputMaterial')
    bsdf = nodes.new('ShaderNodeBsdfPrincipled')
    bsdf.inputs['Base Color'].default_value = base_color
    bsdf.inputs['Metallic'].default_value = metallic
    bsdf.inputs['Roughness'].default_value = roughness

    mat.node_tree.links.new(bsdf.outputs['BSDF'], output.inputs['Surface'])
    output.location = (300, 0)
    bsdf.location = (0, 0)
    return mat
```