# Lighting Reference: AAA Game & Environment Lighting

## Philosophy
Game lighting is fundamentally different from film/arch-viz: **it must look good at runtime, not just in renders**. Every lighting decision must consider:
- Baked vs dynamic lighting budget
- HDRI → Skylight conversion for UE5
- Mood consistency across time-of-day variants

---

## Production Lighting Setups

### 1. Hero Asset Presentation Rig (KitBash/Fab Portfolio)
Best for showcasing a single hero prop or kit piece.

```python
import bpy, math

def create_hero_presentation_rig(target_obj_name=None):
    """
    Production-quality 3-point rig for asset presentation.
    Mimics KitBash3D/Fab showcase lighting.
    """
    # Get scene center or target object
    if target_obj_name and target_obj_name in bpy.data.objects:
        target = bpy.data.objects[target_obj_name]
        center = target.location.copy()
        size = max(target.dimensions)
    else:
        center = (0, 0, 0)
        size = 2.0

    dist = size * 3.0  # Camera/light distance relative to asset size

    # Collection setup
    lgt_col = bpy.data.collections.get("LGT_Setup") or bpy.data.collections.new("LGT_Setup")
    if "LGT_Setup" not in bpy.context.scene.collection.children:
        bpy.context.scene.collection.children.link(lgt_col)

    def add_area_light(name, location, rotation_euler, energy, color_temp_k, size):
        bpy.ops.object.light_add(type='AREA', location=location)
        light_obj = bpy.context.active_object
        light_obj.name = name
        light_obj.rotation_euler = rotation_euler
        light = light_obj.data
        light.energy = energy
        light.color = kelvin_to_rgb(color_temp_k)
        light.size = size
        light.use_shadow = True
        light.shadow_soft_size = size * 0.5
        # Move to LGT collection
        for col in light_obj.users_collection:
            col.objects.unlink(light_obj)
        lgt_col.objects.link(light_obj)
        return light_obj

    # Key Light: warm, dominant, 45° above and to the side
    key = add_area_light(
        "LGT_Key",
        location=(dist * 0.7, -dist * 0.7, dist * 0.8),
        rotation_euler=(math.radians(45), 0, math.radians(45)),
        energy=1000 * size,
        color_temp_k=5500,
        size=size * 1.5
    )

    # Fill Light: cooler, soft, opposite side
    fill = add_area_light(
        "LGT_Fill",
        location=(-dist * 0.8, -dist * 0.3, dist * 0.3),
        rotation_euler=(math.radians(20), 0, math.radians(-60)),
        energy=300 * size,
        color_temp_k=6800,
        size=size * 2.5
    )

    # Rim Light: high contrast edge definition from behind
    rim = add_area_light(
        "LGT_Rim",
        location=(0, dist * 0.9, dist * 0.6),
        rotation_euler=(math.radians(-40), 0, math.radians(180)),
        energy=700 * size,
        color_temp_k=7200,
        size=size * 0.8
    )

    bpy.ops.object.select_all(action='DESELECT')
    print(f"[LGT] Hero presentation rig created. Key: {key.name}, Fill: {fill.name}, Rim: {rim.name}")

def kelvin_to_rgb(kelvin):
    """Approximate Kelvin temperature to RGB (Tanner Helland algorithm)."""
    temp = kelvin / 100.0
    if temp <= 66:
        r = 1.0
        g = max(0, min(1, (99.4708025861 * math.log(temp) - 161.1195681661) / 255.0))
        b = 0.0 if temp <= 19 else max(0, min(1, (138.5177312231 * math.log(temp - 10) - 305.0447927307) / 255.0))
    else:
        r = max(0, min(1, (329.698727446 * ((temp - 60) ** -0.1332047592)) / 255.0))
        g = max(0, min(1, (288.1221695283 * ((temp - 60) ** -0.0755148492)) / 255.0))
        b = 1.0
    return (r, g, b)

create_hero_presentation_rig()
```

### 2. Environment Lighting: Exterior Day

```python
import bpy

def setup_exterior_day(hdri_path=None):
    """
    Production exterior day setup.
    If hdri_path provided, loads it. Otherwise sets up sun + sky.
    """
    world = bpy.context.scene.world
    world.use_nodes = True
    nodes = world.node_tree.nodes
    links = world.node_tree.links
    nodes.clear()

    if hdri_path:
        # HDRI path
        tex_coord = nodes.new('ShaderNodeTexCoord')
        mapping = nodes.new('ShaderNodeMapping')
        env_tex = nodes.new('ShaderNodeTexEnvironment')
        background = nodes.new('ShaderNodeBackground')
        output = nodes.new('ShaderNodeOutputWorld')

        env_tex.image = bpy.data.images.load(hdri_path)
        background.inputs['Strength'].default_value = 1.0

        links.new(tex_coord.outputs['Generated'], mapping.inputs['Vector'])
        links.new(mapping.outputs['Vector'], env_tex.inputs['Vector'])
        links.new(env_tex.outputs['Color'], background.inputs['Color'])
        links.new(background.outputs['Background'], output.inputs['Surface'])
    else:
        # Sky Texture fallback
        sky = nodes.new('ShaderNodeTexSky')
        sky.sky_type = 'HOSEK_WILKIE'
        sky.sun_elevation = 0.5  # ~30 degrees
        sky.sun_rotation = 0.3
        sky.turbidity = 2.5
        background = nodes.new('ShaderNodeBackground')
        output = nodes.new('ShaderNodeOutputWorld')
        background.inputs['Strength'].default_value = 1.5
        links.new(sky.outputs['Color'], background.inputs['Color'])
        links.new(background.outputs['Background'], output.inputs['Surface'])

    # Add directional sun to match HDRI/sky
    bpy.ops.object.light_add(type='SUN', location=(0, 0, 10))
    sun = bpy.context.active_object
    sun.name = "LGT_Sun_Key"
    sun.data.energy = 3.0
    sun.data.angle = 0.00872665  # ~0.5 degrees, sharp sun

setup_exterior_day()
```

---

## Light Color Temperature Cheatsheet

| Kelvin | Character | Use Case |
|--------|-----------|----------|
| 2700K | Very warm orange | Candles, fire, sunset |
| 3200K | Warm white | Tungsten practical, golden hour |
| 4000K | Neutral warm | Interior fill, overcast fill |
| 5500K | Daylight neutral | Midday sun key light |
| 6500K | Cool white | Overcast sky, shadow areas |
| 7500K+ | Blue-cool | Night fill, moonlight, sci-fi |

## Baking Considerations for Game Assets
- Use **UV Channel 1** (second UV) for lightmap baking — non-overlapping, margins 4–8px
- Bake `Combined` for hero props, `Diffuse` + `Shadow` separately for flexibility
- Lightmap resolution: 512×512 minimum, 2048×2048 for hero props
- In UE5: import with "Generate Lightmap UVs" OFF — always bring your own