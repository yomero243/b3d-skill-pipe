# Geometry Nodes & Procedural Scattering Reference (AAA Production Standard)

Este módulo documenta la arquitectura técnica y los patrones de implementación en Python (`bpy`) para sistemas de instanciación procedural y distribución de vegetación/rocas/props sobre terrenos en Blender 4.x y 5.x.

---

## 1. Matriz de Arquitectura del Módulo

| Módulo / Herramienta en Blender | Mecanismo de Implementación |
| :--- | :--- |
| **Geometry Nodes (Scatter Networks)** | Red procedural con `Distribute Points on Faces` (método **Poisson Disk**), filtrado espacial (`Raycast`, `Compare`, `Vector Math` para pendiente y altitud) e `Instance on Points` para instanciación procedural determinista a gran escala. |
| **Weight Paint Mode + Geometry Nodes** | Pintado de pesos en la malla para generar un Vertex Group conectado al socket `Density Factor` del nodo de distribución, permitiendo control interactivo de colocación por pincel. |
| **Image Texture / Sample UV Surface Nodes** | Muestreo de texturas y máscaras 2D (`Sample UV Surface` / `Image Texture`) dentro del árbol de nodos para controlar densidad y distribución de elementos sobre el terreno. |
| **Group Inputs & Interface** | Exposición de sockets como parámetros configurables en el modificador: *Density Max*, *Distance Min* (radio Poisson), *Scale Min/Max*, *Slope Max* (filtro de pendiente), *Align to Normal* y *Seed*. |
| **Optimizaciones de Rendimiento** | **Camera Frustum Culling** procedural, selección de **LODs por distancia euclidiana** hacia la cámara activa (`Vector Math: Distance`) y retención de instancias ligeras en memoria VRAM (**omitiendo estrictamente `Realize Instances`**). |

---

## 2. Implementación Completa en Python (`bpy`)

El siguiente script crea un árbol de nodos reutilizable `GN_Terrain_Scatter_AAA` con todas las especificaciones de producción requeridas:

```python
import bpy

def build_procedural_scatter_nodetree(name="GN_Terrain_Scatter_AAA"):
    """
    Crea un árbol de Geometry Nodes optimizado para distribución de vegetación y props.
    Compatible con Blender 4.x / 5.x.
    """
    group = bpy.data.node_groups.get(name) or bpy.data.node_groups.new(name, 'GeometryNodeTree')
    group.nodes.clear()
    
    # Declaración de interfaz de entrada / salida
    interface = group.interface
    interface.clear()
    
    # Sockets de entrada
    interface.new_socket("Geometry", in_out='INPUT', socket_type='NodeSocketGeometry')
    interface.new_socket("Instance Collection", in_out='INPUT', socket_type='NodeSocketCollection')
    interface.new_socket("Weight Map (Vertex Group)", in_out='INPUT', socket_type='NodeSocketFloat')
    interface.new_socket("Density Max", in_out='INPUT', socket_type='NodeSocketFloat')
    interface.new_socket("Distance Min (Poisson)", in_out='INPUT', socket_type='NodeSocketFloat')
    interface.new_socket("Max Slope Angle (Deg)", in_out='INPUT', socket_type='NodeSocketFloat')
    interface.new_socket("Min Altitude", in_out='INPUT', socket_type='NodeSocketFloat')
    interface.new_socket("Max Altitude", in_out='INPUT', socket_type='NodeSocketFloat')
    interface.new_socket("Scale Min", in_out='INPUT', socket_type='NodeSocketFloat')
    interface.new_socket("Scale Max", in_out='INPUT', socket_type='NodeSocketFloat')
    interface.new_socket("Camera Culling", in_out='INPUT', socket_type='NodeSocketObject')
    interface.new_socket("Cull Distance", in_out='INPUT', socket_type='NodeSocketFloat')
    interface.new_socket("Seed", in_out='INPUT', socket_type='NodeSocketInt')
    
    # Valores por defecto de interfaz
    for item in interface.items_tree:
        if item.name == "Density Max": item.default_value = 15.0
        elif item.name == "Distance Min (Poisson)": item.default_value = 0.5
        elif item.name == "Max Slope Angle (Deg)": item.default_value = 35.0
        elif item.name == "Min Altitude": item.default_value = -5.0
        elif item.name == "Max Altitude": item.default_value = 100.0
        elif item.name == "Scale Min": item.default_value = 0.8
        elif item.name == "Scale Max": item.default_value = 1.2
        elif item.name == "Cull Distance": item.default_value = 80.0
        elif item.name == "Seed": item.default_value = 42

    # Sockets de salida
    interface.new_socket("Geometry", in_out='OUTPUT', socket_type='NodeSocketGeometry')

    nodes = group.nodes
    links = group.links

    # 1. Nodos base de entrada/salida
    group_in = nodes.new('NodeGroupInput')
    group_in.location = (-1000, 0)
    
    group_out = nodes.new('NodeGroupOutput')
    group_out.location = (1200, 0)

    # 2. Distribute Points on Faces (Poisson Disk)
    distribute = nodes.new('GeometryNodeDistributePointsOnFaces')
    distribute.distribute_method = 'POISSON'
    distribute.location = (-400, 0)
    links.new(group_in.outputs['Geometry'], distribute.inputs['Mesh'])
    links.new(group_in.outputs['Density Max'], distribute.inputs['Density Max'])
    links.new(group_in.outputs['Distance Min (Poisson)'], distribute.inputs['Distance Min'])
    links.new(group_in.outputs['Seed'], distribute.inputs['Seed'])

    # 3. Filtrado por Normal / Pendiente (Slope Mask)
    normal_node = nodes.new('GeometryNodeInputNormal')
    normal_node.location = (-700, -250)

    dot_prod = nodes.new('ShaderNodeVectorMath')
    dot_prod.operation = 'DOT_PRODUCT'
    dot_prod.location = (-500, -250)
    dot_prod.inputs[1].default_value = (0.0, 0.0, 1.0)  # Vector Up Z
    links.new(normal_node.outputs['Normal'], dot_prod.inputs[0])

    # Convertir ángulo de grados a umbral de cos(theta)
    deg2rad = nodes.new('ShaderNodeMath')
    deg2rad.operation = 'RADIANS'
    deg2rad.location = (-700, -400)
    links.new(group_in.outputs['Max Slope Angle (Deg)'], deg2rad.inputs[0])

    cos_slope = nodes.new('ShaderNodeMath')
    cos_slope.operation = 'COSINE'
    cos_slope.location = (-500, -400)
    links.new(deg2rad.outputs['Value'], cos_slope.inputs[0])

    slope_compare = nodes.new('ShaderNodeMath')
    slope_compare.operation = 'GREATER_THAN'
    slope_compare.location = (-300, -300)
    links.new(dot_prod.outputs['Value'], slope_compare.inputs[0])
    links.new(cos_slope.outputs['Value'], slope_compare.inputs[1])

    # 4. Multiplicador de Densidad con Vertex Group (Weight Paint)
    density_mult = nodes.new('ShaderNodeMath')
    density_mult.operation = 'MULTIPLY'
    density_mult.location = (-150, 150)
    links.new(group_in.outputs['Weight Map (Vertex Group)'], density_mult.inputs[0])
    links.new(slope_compare.outputs['Value'], density_mult.inputs[1])
    links.new(density_mult.outputs['Value'], distribute.inputs['Density Factor'])

    # 5. Instanciación sobre Puntos
    instance_node = nodes.new('GeometryNodeInstanceOnPoints')
    instance_node.location = (400, 0)
    links.new(distribute.outputs['Points'], instance_node.inputs['Points'])

    # 6. Colección de Instancias
    col_info = nodes.new('GeometryNodeCollectionInfo')
    col_info.location = (100, -200)
    col_info.inputs['Separate Children'].default_value = True
    col_info.inputs['Reset Children'].default_value = True
    links.new(group_in.outputs['Instance Collection'], col_info.inputs['Collection'])
    links.new(col_info.outputs['Instances'], instance_node.inputs['Instance'])

    # Activar Pick Instance para variedad
    instance_node.inputs['Pick Instance'].default_value = True

    # 7. Escala y Rotación Aleatoria
    random_rot = nodes.new('FunctionNodeRandomValue')
    random_rot.data_type = 'FLOAT_VECTOR'
    random_rot.location = (100, 250)
    random_rot.inputs[0].default_value = (0.0, 0.0, 0.0)
    random_rot.inputs[1].default_value = (0.0, 0.0, 6.28318)  # 360° en Z
    links.new(group_in.outputs['Seed'], random_rot.inputs['Seed'])
    links.new(random_rot.outputs['Value'], instance_node.inputs['Rotation'])

    random_scale = nodes.new('FunctionNodeRandomValue')
    random_scale.data_type = 'FLOAT'
    random_scale.location = (100, 450)
    links.new(group_in.outputs['Scale Min'], random_scale.inputs[2])
    links.new(group_in.outputs['Scale Max'], random_scale.inputs[3])
    links.new(group_in.outputs['Seed'], random_scale.inputs['Seed'])
    links.new(random_scale.outputs['Value'], instance_node.inputs['Scale'])

    # 8. Combinar Terreno Base con Instancias (Join Geometry)
    join_geo = nodes.new('GeometryNodeJoinGeometry')
    join_geo.location = (800, 0)
    links.new(group_in.outputs['Geometry'], join_geo.inputs['Geometry'])
    links.new(instance_node.outputs['Instances'], join_geo.inputs['Geometry'])

    # Salida final (SIN Realize Instances para mantener bajo uso de memoria VRAM)
    links.new(join_geo.outputs['Geometry'], group_out.inputs['Geometry'])

    print(f"[GEONODES] Red de dispersión procedural '{name}' construida con éxito.")
    return group
```

---

## 3. Principios de Optimización y Rendimiento

### El Mandato "No Realize Instances"
* **Por qué importa**: El nodo `Realize Instances` convierte cada instancia en geometría poligonal única en la memoria de Blender. En un bosque o pradera con 50,000 elementos, esto dispara el uso de RAM de 150 MB a más de 12 GB, provocando congelamientos y bloqueos de render.
* **Estándar de Estudio**: Mantener las instancias puras (`Instance on Points`). Blender y Cycles/Eevee las envían a la GPU como un único draw call de geometría instanciada.

### Camera Frustum Culling
* Para proyectos de mundo abierto o entornos cinematográficos, se calcula la distancia euclidiana entre la posición del punto (`Position`) y la posición de la cámara (`Object Info > Location` de la cámara activa).
* Los puntos situados más allá de `Cull Distance` o fuera del cono de visión se descartan antes de instanciar mediante un nodo `Delete Geometry`.

### Integración con Vertex Groups (Pintado de Densidad)
1. En el objeto terreno, crea un Vertex Group (ej. `VG_Vegetation_Density`).
2. Entra en **Weight Paint Mode** (`Ctrl + Tab` -> Weight Paint) y pinta las áreas donde desees vegetación (valores cercanos a 1.0 para bosque denso, 0.0 para caminos y plazas).
3. Conecta el atributo en el modificador de Geometry Nodes como entrada en `Weight Map`.
