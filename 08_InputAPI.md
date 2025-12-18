# 8. Input API

The Input API provides a unified, deterministic interface for interacting with players. Whether the user is playing on a keyboard, a touchscreen overlay, or a high-end Bluetooth controller, your code sees a standardized **Virtual Device**.

## 8.1 Polling Philosophy

Input state is captured **once per frame**, immediately before `update(dt)` is called.
*   **Consistency:** The state of a button cannot change *during* an update tick. `input.btn()` will return the same value at the top and bottom of your function.
*   **Determinism:** Recorded input streams can be replayed later to reproduce the exact same gameplay session.

## 8.2 Virtual Gamepads

The console supports up to **4 Players** (Indices 0–3). Each player has a virtual controller equipped with:
*   **D-Pad**: Up, Down, Left, Right.
*   **Face Buttons**: A (South), B (East), X (West), Y (North).
*   **Shoulders**: L1, R1 (Bumpers).
*   **Triggers**: L2, R2 (Analog).
*   **Sticks**: Left Stick, Right Stick (2-axis Analog).
*   **System**: Start, Select (Back).

### Button Constants
These constants are used with button query functions.

| Constant | Description |
| :--- | :--- |
| `input.BTN_A`, `BTN_B`, `BTN_X`, `BTN_Y` | Face buttons. |
| `input.BTN_UP`, `DOWN`, `LEFT`, `RIGHT` | Directional pad. |
| `input.BTN_L1`, `BTN_R1` | Shoulder bumpers. |
| `input.BTN_L2`, `BTN_R2` | Triggers (Digital threshold). |
| `input.BTN_L3`, `BTN_R3` | Stick clicks. |
| `input.BTN_START`, `BTN_SELECT` | Menu buttons. |

### Digital Button Queries

| Function | Description |
| :--- | :--- |
| `input.btn(player, id)` | Returns `true` if the button is currently **held down**. |
| `input.btnp(player, id)` | Returns `true` only on the **frame it was pressed** (rising edge). |
| `input.btnr(player, id)` | Returns `true` only on the **frame it was released** (falling edge). |

**Auto-Repeat for UI:**
`input.btnp` accepts optional timing arguments for menu navigation.
```c
// Returns true on press, then every 0.1s after holding for 0.4s
if (input.btnp(0, input.BTN_DOWN, 0.4, 0.1)) {
    menuSelection++;
}
```

### Analog Axis Queries
Axes return a float value between `-1.0` and `1.0`.
*   **Deadzone:** The runtime applies a default deadzone (0.2) to prevent drift. You can access raw values if needed.

| Constant | Range | Description |
| :--- | :--- | :--- |
| `input.AXIS_LX`, `AXIS_LY` | -1.0 to 1.0 | Left Stick (X=Right, Y=Down). |
| `input.AXIS_RX`, `AXIS_RY` | -1.0 to 1.0 | Right Stick. |
| `input.AXIS_LT`, `AXIS_RT` | 0.0 to 1.0 | Analog Triggers (Idle is 0.0). |

```c
local dx = input.axis(0, input.AXIS_LX);
local dy = input.axis(0, input.AXIS_LY);
// Apply a radial deadzone for precise movement
if (math.sqrt(dx*dx + dy*dy) < 0.2) { dx=0; dy=0; }
```

## 8.3 Mouse Input

The mouse is treated as a shared device (Player 0).
*   **Coordinates:** Automatically scaled to the logical resolution (`MODE_4_3` or `MODE_16_9`). `(0,0)` is always Top-Left.

```c
local m = input.mouse();
// m.x, m.y       : Absolute position
// m.dx, m.dy     : Delta movement since last frame
// m.scrollX, m.scrollY : Wheel scroll
// m.buttons      : Bitmask (1=Left, 2=Right, 4=Middle)
```

### Mouse Queries
You can also use standard button functions with mouse constants:
*   `input.BTN_MOUSE_LEFT`
*   `input.BTN_MOUSE_RIGHT`
*   `input.BTN_MOUSE_MIDDLE`

```c
if (input.btnp(0, input.BTN_MOUSE_LEFT)) {
    fireWeapon(m.x, m.y);
}
```

### Mouse Lock (Capture)
For First-Person games or custom camera controls, you can lock the cursor.
```c
input.setMouseLock(true);
// Cursor becomes invisible and locked to center.
// Use m.dx / m.dy for look controls.
```

## 8.4 Keyboard Input

While gamepads are preferred for portability, full keyboard access is available.
Keys are referenced by **Scancodes** (physical location), ensuring WASD works on QWERTY, AZERTY, and Dvorak layouts without remapping.

Constants are named `input.KEY_A`, `input.KEY_SPACE`, `input.KEY_LEFT`, etc.

```c
if (input.key(input.KEY_LSHIFT) && input.keyp(input.KEY_SPACE)) {
    // Shift+Jump
}
```

**Text Entry:**
Do not use `input.key()` for typing names or chat. Use the `onTextInput` event callback instead, which handles Shift states, Caps Lock, and OS-specific composition (accents).

```c
// In your script
function onTextInput(char) {
    playerName += char;
}
```

## 8.5 Haptics (Rumble)

You can trigger vibration on supported controllers.

```c
// Player, LowFreq (0-1), HighFreq (0-1), Duration (seconds)
input.rumble(0, 0.8, 0.2, 0.5); // Heavy explosion
input.rumble(0, 0.0, 0.5, 0.1); // Light tick
```

## 8.6 Example: Twin-Stick Control Scheme

This example implements a standard control scheme where the Left Stick moves and the Right Stick aims/shoots. It gracefully falls back to Mouse/Keyboard if no gamepad is active.

```c
local player = { x=400, y=300, aimAngle=0 };

function update(dt) {
    // 1. MOVEMENT (Left Stick or WASD)
    local moveX = input.axis(0, input.AXIS_LX);
    local moveY = input.axis(0, input.AXIS_LY);
    
    // Keyboard fallback
    if (input.key(input.KEY_W)) moveY -= 1;
    if (input.key(input.KEY_S)) moveY += 1;
    if (input.key(input.KEY_A)) moveX -= 1;
    if (input.key(input.KEY_D)) moveX += 1;
    
    // Normalize vector
    local len = math.sqrt(moveX*moveX + moveY*moveY);
    if (len > 1.0) { moveX /= len; moveY /= len; }
    
    player.x += moveX * 200 * dt;
    player.y += moveY * 200 * dt;

    // 2. AIMING (Right Stick or Mouse)
    local aimX = input.axis(0, input.AXIS_RX);
    local aimY = input.axis(0, input.AXIS_RY);
    
    if (math.abs(aimX) > 0.1 || math.abs(aimY) > 0.1) {
        // Gamepad aiming
        player.aimAngle = math.atan2(aimY, aimX);
        // Fire automatically when pushing stick
        shoot();
    } else {
        // Mouse aiming
        local m = input.mouse();
        local dx = m.x - player.x;
        local dy = m.y - player.y;
        player.aimAngle = math.atan2(dy, dx);
        
        if (input.btn(0, input.BTN_MOUSE_LEFT)) {
            shoot();
        }
    }
}
```