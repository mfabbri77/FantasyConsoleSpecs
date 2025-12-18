# 6. Graphics 3D API

The Fantasy Console includes a fully-featured 3D rendering engine. Just like the 2D engine, it operates in **Retained Mode**. You construct a scene graph of nodes (meshes, lights, cameras), and the runtime handles the complex pipeline of culling, sorting, lighting, and post-processing.

The 3D coordinate system is **Right-Handed**:
*   **+Y**: Up
*   **+X**: Right
*   **-Z**: Forward (into the screen)
*   **Units**: Arbitrary World Units (1.0 = 1 meter is a good convention).

## 6.1 The Scene Graph

The core of the 3D engine is the **Node Hierarchy**. A scene is a tree of nodes starting from a root.

### Node3D
The base class for all 3D objects. It handles spatial transformation.

| Property | Type | Description |
| :--- | :--- | :--- |
| `position` | `Vec3` | Translation relative to parent. |
| `rotation` | `Quat` | Orientation. Use helper methods to manipulate. |
| `scale` | `Vec3` | Scale factors. Default is `(1,1,1)`. |
| `visible` | `bool` | If false, the node and its children are skipped. |

**Methods:**
*   `node.addChild(childNode)` / `node.removeChild(childNode)`
*   `node.setRotation(eulerVec3)`: Sets rotation from Euler angles (degrees).
*   `node.lookAt(targetVec3, [upVec3])`: Rotates the node to face a point.
*   `node.worldPosition()`: Returns the calculated absolute position.

## 6.2 Meshes (Geometry)

A `Mesh` is an immutable resource defining geometry (vertices, normals, UVs).

### Built-in Primitives
Quickly create standard shapes for prototyping.
```c
local cube   = gfx3d.Mesh.createBox(1.0, 1.0, 1.0);
local sphere = gfx3d.Mesh.createSphere(0.5, 32, 16); // radius, slices, stacks
local plane  = gfx3d.Mesh.createPlane(10.0, 10.0);   // width, depth
```

### Custom Geometry
Create meshes from raw data arrays.
```c
local customMesh = gfx3d.Mesh.fromArrays({
    vertices = [Vec3(-1,0,0), Vec3(1,0,0), Vec3(0,1,0)],
    normals  = [Vec3(0,0,1), Vec3(0,0,1), Vec3(0,0,1)],
    uvs      = [Vec2(0,0), Vec2(1,0), Vec2(0.5,1)],
    indices  = [0, 1, 2]
});
```

### MeshNode
To render a mesh, wrap it in a `MeshNode` and add it to the scene.
```c
local node = gfx3d.MeshNode(cube, material);
scene.root.addChild(node);
```

## 6.3 Materials (PBR Shading)

The engine uses a **Physically Based Rendering (PBR)** workflow (Metallic-Roughness), compatible with glTF 2.0. This ensures materials behave realistically under different lighting conditions.

A `Material` is immutable. Use the builder pattern to create one.

```c
local mat = gfx3d.Material()
    .baseColor(Color(1.0, 0.0, 0.0, 1.0)) // Red
    .metallic(0.0)      // 0.0 = Dielectric (Plastic/Wood), 1.0 = Metal
    .roughness(0.5)     // 0.0 = Smooth Mirror, 1.0 = Matte
    .emissive(Color(0,0,0,1)) // Self-illumination color
    .cullMode(gfx3d.CullMode.BACK); // BACK, FRONT, NONE
```

### Textures
You can map properties to textures for detail.
```c
mat.albedoMap(gfx.Image.load("assets/wood_diffuse.png"))
   .normalMap(gfx.Image.load("assets/wood_normal.png"))
   .roughnessMap(gfx.Image.load("assets/wood_rough.png"));
```

## 6.4 Lighting

Lighting is dynamic and calculated per-pixel.

### LightNode
A node that emits light.
*   **Directional:** Infinite light (Sun). Position is ignored, only rotation matters.
*   **Point:** Emits in all directions from a point. Has a `range`.
*   **Spot:** Emits in a cone. Has `range`, `innerAngle`, `outerAngle`.

```c
local sun = gfx3d.LightNode(gfx3d.LightType.DIRECTIONAL);
sun.color = Color(1.0, 0.9, 0.8, 1.0);
sun.intensity = 1.5;
sun.lookAt(Vec3(0, 0, 0)); // Shine towards origin
scene.root.addChild(sun);
```

### Ambient Light
Set globally on the `Scene3D` object. It lifts the darkest shadows.
```c
scene.environment.ambientColor = Color(0.1, 0.1, 0.2, 1.0);
```

## 6.5 Cameras

The `CameraNode` defines the viewport.

```c
local cam = gfx3d.CameraNode();
cam.setPerspective(60, 0.1, 1000.0); // FOV (deg), Near, Far
// OR
cam.setOrthographic(10.0, 0.1, 1000.0); // Size (vertical units), Near, Far

cam.position = Vec3(0, 5, 10);
cam.lookAt(Vec3(0, 0, 0));

scene.activeCamera = cam;
```

## 6.6 Rendering the Scene

To draw 3D content, you create a `Scene3D` container, populate it, and then submit it via `gfx3d.draw()`.

**Integration with 2D:**
The 3D render occurs immediately when called. It writes to the current 2D framebuffer (or an offscreen target). This allows you to mix 2D and 3D:
1.  Draw 2D Background.
2.  Draw 3D Scene.
3.  Draw 2D HUD on top.

```c
function draw() {
    // 1. 2D Background
    local ctx = gfx.beginFrame();
    ctx.clear(Color(0,0,0,1));
    gfx.drawScene(ctx);

    // 2. 3D Scene
    // (Assuming 'myScene3D' was built in init())
    gfx3d.draw(myScene3D);

    // 3. 2D UI
    local ui = gfx.beginFrame();
    ui.drawText("Score: 0", 10, 10, uiStyle);
    gfx.drawScene(ui);
}
```

## 6.7 Post-Processing (FX)

The 3D engine supports a stack of screen-space effects. These are configured on the `Scene3D` object.

```c
myScene3D.postProcess = {
    // Bloom: Glowing highlights
    bloom = {
        enabled = true,
        threshold = 0.8,
        intensity = 1.2,
        radius = 0.5
    },
    // Tone Mapping: Converting HDR light to screen colors
    toneMapping = gfx3d.ToneMapping.ACES, // REINHARD, LINEAR, ACES
    
    // Color Grading
    colorGrading = {
        saturation = 1.2,
        contrast = 1.1,
        exposure = 1.0
    }
};
```

## 6.8 Animation Integration

You can use the **Chapter 5** animation system to animate 3D nodes. The property handles work seamlessly.

```c
local anim = gfx.anim.create("spin_cube");
anim.track(anim.prop(cubeNode).rotation, [
    { time=0, value=Quat.fromEuler(0,0,0) },
    { time=2, value=Quat.fromEuler(0,360,0) }
]);
anim.play({ loop=true });
```

## 6.9 Example: A Complete 3D Scene

```c
local scene3d;
local cubeNode;

function init() {
    // 1. Create Scene
    scene3d = gfx3d.Scene();

    // 2. Camera
    local cam = gfx3d.CameraNode();
    cam.setPerspective(45, 0.1, 100.0);
    cam.position = Vec3(3, 3, 3);
    cam.lookAt(Vec3(0, 0, 0));
    scene3d.activeCamera = cam;
    scene3d.root.addChild(cam);

    // 3. Lights
    local light = gfx3d.LightNode(gfx3d.LightType.POINT);
    light.position = Vec3(2, 5, 2);
    light.color = Color(1, 0.8, 0.6, 1);
    light.intensity = 5.0;
    light.range = 20.0;
    scene3d.root.addChild(light);

    // 4. Object
    local mesh = gfx3d.Mesh.createBox();
    local mat = gfx3d.Material()
        .baseColor(Color(0.2, 0.4, 0.8, 1))
        .roughness(0.2)
        .metallic(0.8);
    
    cubeNode = gfx3d.MeshNode(mesh, mat);
    scene3d.root.addChild(cubeNode);
}

function update(dt) {
    // Rotate cube manually (or use animation system)
    local q = Quat.fromEuler(0, dt * 90, 0); // 90 deg/sec
    cubeNode.rotation = cubeNode.rotation * q;
}

function draw() {
    // Clear screen first
    local bg = gfx.beginFrame();
    bg.clear(Color(0.05, 0.05, 0.05, 1));
    gfx.drawScene(bg);

    // Draw 3D
    gfx3d.draw(scene3d);
}
```