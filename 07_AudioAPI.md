# 7. Audio API

The Fantasy Console audio subsystem is built on two pillars: **Tracker Modules** for complex, dynamic background music, and a **Polyphonic Sample Engine** for high-fidelity sound effects.

Crucially, the entire audio stack is **Deterministic**.
*   Audio processing happens in lockstep with the `update()` loop.
*   If you play a sound on frame 100, it will always start exactly on frame 100.
*   The internal mixer prevents "drift" between gameplay logic and audio playback, making this platform ideal for rhythm games.

## 7.1 Music (Tracker Modules)

For music, the console supports standard tracker formats: **.MOD, .XM, .S3M, .IT**.
Tracker modules are preferred over streamed audio (MP3/OGG) because they are tiny, loop perfectly, and allow for dynamic programmatic control of individual channels (muting tracks, changing tempo).

### Loading and Playback

```c
// Load a module (Synchronous, Cached)
local bgm = audio.loadModule("assets/music/stage1.xm");

// Start playback
// loop: boolean
// startOrder: pattern order index to start at (default 0)
audio.playMusic(bgm, true, 0);
```

### Music Control

Global controls for the currently playing module:

| Function | Description |
| :--- | :--- |
| `audio.pauseMusic()` | Pauses playback. |
| `audio.resumeMusic()` | Resumes from the paused point. |
| `audio.stopMusic()` | Stops and resets position to the beginning. |
| `audio.setMusicVolume(vol)` | Master volume for music (0.0 to 1.0). |
| `audio.setMusicTempo(bpm)` | Overrides the module's BPM. Pass `null` to reset to default. |
| `audio.seekMusic(order, row)` | Jumps instantly to a specific pattern order and row. |

### Synchronization Callbacks
To build rhythm mechanics, you need to know exactly when a beat hits. You can register a callback that triggers on every row change.

```c
audio.onMusicRow(function(row, order, pattern) {
    // Flash the screen on every 4th row (typical 4/4 beat)
    if (row % 4 == 0) {
        screenFlash = 1.0;
    }
});
```

## 7.2 Sound Effects (SFX)

The SFX system handles short, transient audio samples (WAV, OGG). The engine supports up to **32 concurrent voices**. If the limit is reached, the oldest or quietest voice is virtually stolen based on priority.

### Loading Samples

```c
local jumpSnd = audio.loadSample("assets/sfx/jump.wav");
local engineSnd = audio.loadSample("assets/sfx/engine_loop.ogg");
```

### Playing Sounds & Voices
When you play a sample, the system returns a **Voice ID** (integer). This ID is a "handle" to the active instance of that sound, allowing you to manipulate it while it plays (e.g., for Doppler effects).

```c
// Basic Play
audio.play(jumpSnd);

// Advanced Play with Parameters
local voiceId = audio.play(engineSnd, {
    volume = 0.8,   // 0.0 to 1.0
    pan = 0.0,      // -1.0 (Left) to 1.0 (Right)
    pitch = 1.0,    // Playback rate (0.5 = half speed, 2.0 = double)
    loop = true,    // Loop indefinitely
    priority = 10   // Higher numbers resist being stolen
});
```

### Manipulating Active Voices
Using the `voiceId`, you can modify properties in real-time inside your `update()` loop.

| Function | Description |
| :--- | :--- |
| `audio.voice.stop(id)` | Stops this specific voice immediately. |
| `audio.voice.setVolume(id, vol)` | Changes volume (smoothly ramped to avoid clicks). |
| `audio.voice.setPitch(id, pitch)` | Changes pitch/speed. |
| `audio.voice.setPan(id, pan)` | Changes stereo panning. |
| `audio.voice.isValid(id)` | Returns `true` if the voice is still playing. |

## 7.3 The Master Mixer

The mixer controls the final summation of Music and SFX. It includes a limiter to prevent digital clipping (distortion) when many loud sounds play at once.

```c
audio.setMasterVolume(1.0); // Global gain
```

### Ducking (Sidechain)
A common technique is to lower the music volume when important SFX (like dialogue or explosions) play.

```c
// Automatically lower music by 50% (to 0.5) over 0.1s when triggered
audio.duckMusic(0.5, 0.1);

// Restore music to normal over 0.5s
audio.unduckMusic(0.5);
```

## 7.5 Audio Synthesis (`audio.Synth`)

To support procedural audio generation (common in 4k demos), the console provides a basic synthesizer.

```c
// Create a synthesizer voice
// Types: SINE, SQUARE, SAW, TRIANGLE, NOISE
local osc = audio.Synth(audio.Wave.SAW);

// ADSR Envelope: Attack, Decay, Sustain, Release
osc.setEnvelope(0.01, 0.1, 0.5, 0.2); 

// Play frequency
// osc, frequency (Hz), duration (sec)
audio.playTone(osc, 440.0, 1.0); 
```

## 7.6 Example: Dynamic Car Engine

This example demonstrates how to use `VoiceID` to modulate a looping engine sound based on game physics.

```c
local engineSample;
local engineVoice = -1; // Invalid ID initially
local carSpeed = 0.0;

function init() {
    engineSample = audio.loadSample("assets/sfx/car_idle.wav");
    
    // Start the loop immediately
    engineVoice = audio.play(engineSample, {
        loop = true,
        volume = 0.0, // Start silent
        pitch = 0.8
    });
}

function update(dt) {
    // Simulate car physics
    if (input.btn(0, input.BTN_A)) {
        carSpeed += dt * 50;
    } else {
        carSpeed -= dt * 30;
    }
    // Clamp speed 0..100
    if (carSpeed < 0) carSpeed = 0;
    if (carSpeed > 100) carSpeed = 100;

    // Modulate Audio based on Speed
    if (audio.voice.isValid(engineVoice)) {
        // Pitch goes from 0.8 (idle) to 2.0 (high revs)
        local targetPitch = 0.8 + (carSpeed / 100.0) * 1.2;
        audio.voice.setPitch(engineVoice, targetPitch);
        
        // Volume fades in when moving
        local targetVol = 0.2 + (carSpeed / 100.0) * 0.8;
        if (carSpeed < 1.0) targetVol = 0.2; // Idle rumble
        audio.voice.setVolume(engineVoice, targetVol);
    }
}
```