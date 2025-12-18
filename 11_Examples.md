# 11. Examples Cookbook

This chapter provides complete, copy-pasteable examples that demonstrate how to combine the various subsystems (Graphics, Audio, Input, Animation) into cohesive applications.

These examples assume a standard project structure with a valid `manifest.ini`.

## 11.1 Example A: Vector Arcade (2D Physics & Input)

This example demonstrates the core **Retained Mode** philosophy:
1.  **Init:** Build geometry (`Path`) and styles once.
2.  **Update:** Mutate the mathematical state (Position, Velocity).
3.  **Draw:** Apply transforms to the geometry based on state.

**Features:** Inertia, Screen Wrapping, Particle System (simulated), Audio triggers.

```c
// --- CONFIGURATION ---
const DRAG = 0.98;
const THRUST = 500.0;
const ROT_SPEED = 4.0;

// --- STATE ---
local ship = {
    pos = Vec2(400, 300),
    vel = Vec2(0, 0),
    angle = 0.0, // Radians
    thrusting = false
};

// --- RESOURCES ---
local shipPath;
local flamePath;
local shipStyle;
local flameStyle;
local sfxThrust;

function init() {
    // 1. Build Geometry (Immutable)
    shipPath = gfx.Path();
    shipPath.moveTo(10, 0);
    shipPath.lineTo(-10, 8);
    shipPath.lineTo(-6, 0);
    shipPath.lineTo(-10, -8);
    shipPath.close();

    flamePath = gfx.Path();
    flamePath.moveTo(-6, 0);
    flamePath.lineTo(-15, 0);

    // 2. Build Styles
    shipStyle = gfx.Style()
        .fill(gfx.Paint.color(0, 0, 0, 1))
        .stroke(gfx.Paint.color(1, 1, 1, 1)) // White outline
        .strokeStyle(gfx.StrokeStyle().width(2));

    flameStyle = gfx.Style()
        .stroke(gfx.Paint.color(1, 0.5, 0, 1)) // Orange
        .strokeStyle(gfx.StrokeStyle().width(2));

    // 3. Audio
    sfxThrust = audio.loadSample("assets/thrust.wav");
}

function update(dt) {
    // Input Handling
    if (input.btn(0, input.BTN_LEFT))  ship.angle -= ROT_SPEED * dt;
    if (input.btn(0, input.BTN_RIGHT)) ship.angle += ROT_SPEED * dt;
    
    ship.thrusting = input.btn(0, input.BTN_UP);

    if (ship.thrusting) {
        // Physics: Acceleration
        ship.vel.x += math.cos(ship.angle) * THRUST * dt;
        ship.vel.y += math.sin(ship.angle) * THRUST * dt;
        
        // Audio: Loop thrust sound if not playing
        // (Implementation dependent on specific audio logic, simplified here)
        if (math.rand() < 0.1) audio.play(sfxThrust, { volume=0.2 });
    }

    // Physics: Integration & Drag
    ship.pos.x += ship.vel.x * dt;
    ship.pos.y += ship.vel.y * dt;
    ship.vel.x *= DRAG;
    ship.vel.y *= DRAG;

    // Screen Wrap (800x600 logic)
    if (ship.pos.x < 0) ship.pos.x += 800;
    if (ship.pos.x > 800) ship.pos.x -= 800;
    if (ship.pos.y < 0) ship.pos.y += 600;
    if (ship.pos.y > 600) ship.pos.y -= 600;
}

function draw() {
    local scene = gfx.beginFrame();
    scene.clear(Color(0.1, 0.1, 0.15, 1)); // Deep space blue

    // Save state before transforming
    scene.save();
    
    // Apply Transform: Translate -> Rotate
    scene.transform(gfx.Transform.translate(ship.pos.x, ship.pos.y));
    scene.transform(gfx.Transform.rotate(math.degrees(ship.angle)));

    // Draw Ship
    scene.drawPath(shipPath, shipStyle);

    // Draw Flame (Conditional)
    if (ship.thrusting) {
        // Flicker effect using deterministic time
        if (time.frame() % 3 != 0) {
            scene.drawPath(flamePath, flameStyle);
        }
    }

    scene.restore();
    gfx.drawScene(scene);
}
```

## 11.2 Example B: Kinetic UI (Animation & Timelines)

This example shows how to build a title screen with sequenced animations and mouse interaction.

**Features:** Property Handles, Timelines, Easing, Mouse Hit Testing.

```c
// --- STATE ---
local logoState = { scale = Vec2(0,0), alpha = 0.0, y = 200 };
local btnState  = { scale = 1.0, color = Color(0.5, 0.5, 0.5, 1) };

// --- RESOURCES ---
local font;
local logoAnim; // Timeline
local hoverAnim; // Animation

function init() {
    font = gfx.Font.load("assets/bold.ttf");

    // 1. Intro Sequence Timeline
    logoAnim = gfx.anim.timeline();
    
    // Pop in the logo (Scale 0 -> 1)
    local trackScale = gfx.anim.create("scale");
    trackScale.track(anim.prop(logoState).scale, [
        { time=0, value=Vec2(0,0) },
        { time=0.8, value=Vec2(1,1), easing=gfx.Easing.elasticOut }
    ]);
    
    // Fade in (Alpha 0 -> 1)
    local trackAlpha = gfx.anim.create("alpha");
    trackAlpha.track(anim.prop(logoState).alpha, [
        { time=0, value=0.0 },
        { time=0.5, value=1.0 }
    ]);

    logoAnim.add(trackScale);
    logoAnim.add(trackAlpha, -0.6); // Overlap
    logoAnim.play();

    // 2. Button Hover Animation
    hoverAnim = gfx.anim.create("hover");
    hoverAnim.track(anim.prop(btnState).scale, [
        { time=0, value=1.0 },
        { time=0.2, value=1.2, easing=gfx.Easing.easeOutQuad }
    ]);
    hoverAnim.track(anim.prop(btnState).color, [
        { time=0, value=Color(0.5, 0.5, 0.5, 1) }, // Grey
        { time=0.2, value=Color(1.0, 0.8, 0.2, 1) } // Gold
    ]);
}

function update(dt) {
    // Mouse Interaction
    local m = input.mouse();
    // Simple box check for button at (400, 400) size 200x50
    local isHover = (math.abs(m.x - 400) < 100 && math.abs(m.y - 400) < 25);

    if (isHover) {
        if (!hoverAnim.isPlaying() && btnState.scale < 1.1) {
            hoverAnim.play({ loop=false });
        }
    } else {
        hoverAnim.stop();
        btnState.scale = 1.0;
        btnState.color = Color(0.5, 0.5, 0.5, 1);
    }
}

function draw() {
    local scene = gfx.beginFrame();
    scene.clear(Color(0,0,0,1));

    // Draw Logo (Animated)
    scene.save();
    scene.transform(gfx.Transform.translate(400, logoState.y));
    scene.transform(gfx.Transform.scale(logoState.scale.x, logoState.scale.y));
    
    local titleStyle = gfx.Style()
        .fill(gfx.Paint.color(1, 1, 1, logoState.alpha));
    local ts = gfx.TextStyle().font(font).size(60).align(gfx.TextAlign.CENTER);
    
    scene.drawText("FANTASY", 0, 0, ts, titleStyle);
    scene.restore();

    // Draw Button (Animated)
    scene.save();
    scene.transform(gfx.Transform.translate(400, 400));
    scene.transform(gfx.Transform.scale(btnState.scale));
    
    local btnPath = gfx.Path();
    btnPath.rect(-100, -25, 200, 50, 10, 10);
    
    scene.drawPath(btnPath, gfx.Style().fill(gfx.Paint.color(btnState.color)));
    
    local btnTs = gfx.TextStyle().font(font).size(24).align(gfx.TextAlign.CENTER);
    scene.drawText("START GAME", 0, 8, btnTs, gfx.Style().fill(gfx.Paint.color(0,0,0,1)));
    
    scene.restore();
    gfx.drawScene(scene);
}
```

## 11.3 Example C: Hybrid World (3D Scene + 2D HUD)

This example demonstrates the power of the engine: rendering a PBR 3D scene and overlaying a crisp vector HUD on top.

**Features:** 3D Scene Graph, Lights, Camera control, 2D/3D Layering.

```c
// --- RESOURCES ---
local scene3d;
local playerNode;
local coinNode;
local hudFont;

function init() {
    // --- 3D SETUP ---
    scene3d = gfx3d.Scene();

    // 1. Camera
    local cam = gfx3d.CameraNode();
    cam.setPerspective(60, 0.1, 100.0);
    cam.position = Vec3(0, 5, 10);
    cam.lookAt(Vec3(0, 0, 0));
    scene3d.activeCamera = cam;
    scene3d.root.addChild(cam);

    // 2. Light
    local sun = gfx3d.LightNode(gfx3d.LightType.DIRECTIONAL);
    sun.lookAt(Vec3(-1, -1, -1));
    sun.intensity = 2.0;
    scene3d.root.addChild(sun);

    // 3. Player (Cube)
    local mesh = gfx3d.Mesh.createBox();
    local mat = gfx3d.Material().baseColor(Color(0, 0.5, 1, 1)).roughness(0.2);
    playerNode = gfx3d.MeshNode(mesh, mat);
    scene3d.root.addChild(playerNode);

    // 4. Coin (Cylinder)
    local coinMesh = gfx3d.Mesh.createCylinder(0.5, 0.1, 16);
    local coinMat = gfx3d.Material()
        .baseColor(Color(1, 0.8, 0, 1))
        .metallic(1.0)
        .roughness(0.1);
    coinNode = gfx3d.MeshNode(coinMesh, coinMat);
    coinNode.position = Vec3(3, 0, 0);
    coinNode.rotation = Quat.fromEuler(90, 0, 0); // Stand up
    scene3d.root.addChild(coinNode);

    // --- 2D SETUP ---
    hudFont = gfx.Font.load("assets/mono.ttf");
}

function update(dt) {
    // Move Player
    local dx = input.axis(0, input.AXIS_LX);
    local dz = input.axis(0, input.AXIS_LY); // Y axis on stick maps to Z in 3D
    
    playerNode.position.x += dx * 5 * dt;
    playerNode.position.z += dz * 5 * dt;

    // Rotate Coin
    local rot = Quat.fromEuler(0, 0, 90 * dt); // Spin Z axis local
    coinNode.rotation = coinNode.rotation * rot;
    
    // Camera Follow (Smooth Lerp)
    local targetPos = playerNode.position + Vec3(0, 5, 10);
    scene3d.activeCamera.position = scene3d.activeCamera.position.lerp(targetPos, dt * 2.0);
}

function draw() {
    // 1. Draw 3D World
    // The 3D engine clears the depth buffer automatically.
    // We clear the color buffer manually via the 2D background first.
    
    local bg = gfx.beginFrame();
    bg.clear(Color(0.5, 0.7, 0.9, 1)); // Sky color
    gfx.drawScene(bg);

    gfx3d.draw(scene3d);

    // 2. Draw 2D HUD Overlay
    local hud = gfx.beginFrame();
    
    local ts = gfx.TextStyle().font(hudFont).size(20);
    local style = gfx.Style().fill(gfx.Paint.color(1, 1, 1, 1));
    
    hud.drawText("P1 POS: " + playerNode.position.toString(), 20, 30, ts, style);
    hud.drawText("COINS: 0", 20, 60, ts, style);

    gfx.drawScene(hud);
}
```