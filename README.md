# Dikarier Effect System API (v2.6)

[Read in Russian](README_RU.md) | [日本語で読む](README_JA.md) | [中文阅读](README_ZH.md)

## Introduction

This documentation is intended for developers who wish to create their own visual effects (addons) or extend the functionality of the **Dikarier Effect System**.

The plugin provides a global interface for manipulating the `PIXI.Container` (game screen) and applying post-processing filters based on States or events.

> **Developer License Agreement:**
> You have the right to create, distribute, and sell your own addons/extensions that utilize this API.
> Resale or redistribution of the source code of the plugin `Dikarier_EffectSystem.js` itself is prohibited.
> The **Dikarier_Core** plugin must be installed in the user's project for visual effects to work.

### ⚠️ Critical: Naming Convention

The plugin's parser uses strict tag filtering to optimize performance.
For your effect to be recognized by the system and added to the processing loop, its name (tag) **MUST** follow this format:

` <DikarierEffectName> `

*   The prefix `DikarierEffect` is mandatory.
*   The name must be in CamelCase (recommended).
*   The tag must end with `>`.

**Examples:**
*   ✅ `<DikarierEffectRadiation>` — Correct. Will be processed.
*   ❌ `<RadiationEffect>` — Ignored (no prefix).
*   ❌ `<DikarierRadiation>` — Ignored (incomplete prefix).
*   ❌ `<MyCustomTag>` — Ignored.

#### Automatic Function Binding
When processing an effect, the system automatically searches for an update function based on the name derived from the tag.
For the tag `<DikarierEffectFire>`, the system will look for the function `updateFireEffect`.

---

## Architecture and Global Access

All functionality is available through the global namespace `des`. The main effect management class is located in `des.mainClass`.

```javascript
// Main effect system class
const EffectSystem = des.mainClass;

// Registry of registered effects
const Registry = des.mainClass.effectRegistry;

// Array of active effects in the current frame
const ActiveEffects = des.mainClass.activeEffects;
```

The system works by intercepting the rendering of `Scene_Base` and modifying the properties of `Spriteset_Map` or `Spriteset_Battle` before the frame is drawn.

---

## `effectData` Object Structure

Each active effect is represented by an `effectData` object. This object is passed to the effect's update function every frame. Understanding this structure is critical for managing the effect's state.

| Property | Type | Description |
| :--- | :--- | :--- |
| `id` | `String` | Unique identifier for the effect instance (generated automatically). |
| `effect` | `String` | Full tag name of the effect (e.g., `<DikarierEffectMyCustom>`). |
| `param` | `Mixed` | Additional parameter. **Important:** When using the standard tag parser `<Tag: X>`, the value `X` is interpreted as `fadeDuration`, and `param` remains `null` (except for the Slow effect). Manual `applyEffect` calls are required to pass custom parameters. |
| `actor` | `Game_Actor` | The actor on whom the effect is applied. |
| `stateId` | `Number` | Database State ID (if the effect is triggered by a state). |
| `totalDuration` | `Number` | Total duration of the effect in frames. `Infinity` if duration depends on a state. |
| `fadeDuration` | `Number` | Duration of the Fade In and Fade Out phases in frames. |
| `elapsed` | `Number` | Number of frames elapsed since the start of the effect. |
| `intensity` | `Number` | Current effect intensity (from `0.0` to `1.0`). Calculated automatically by the system. |
| `state` | `String` | Current phase: `'fadeIn'`, `'full'`, or `'fadeOut'`. |

---

## Creating and Registering an Effect

To add a new effect, you must register an update function in the `effectRegistry`. The function is called every frame for an active effect.

### Update Function Signature

```javascript
/**
 * @param {Object} screen - The PIXI.Container instance (usually Spriteset).
 * @param {Object} effectData - Data object for the current effect.
 * @param {Array} currentFrameFilters - Array of PIXI filters for the current frame.
 */
function updateMyEffect(screen, effectData, currentFrameFilters) {
    // Effect logic
}
```

### Implementation Example

Below is an example of creating an "Underwater" effect, which adds a wave distortion and a blue tint.

```javascript
// Registering the effect tag
des.mainClass.effectRegistry['<DikarierEffectUnderwater>'] = function(screen) {
    // Initialize unique container properties on first run
    screen._underwaterTime = screen._underwaterTime || { time: 0 };
};

// Adding update logic.
// Function name must follow the pattern: update + EffectName + Effect
des.mainClass.updateUnderwaterEffect = function(screen, effectData, currentFrameFilters) {
    // Get global effect time for animation synchronization
    const time = des.mainClass.effectTime * 0.05;
    
    // Get calculated intensity (accounts for fade in/out)
    const intensity = effectData.intensity;

    // Apply screen transformation (swaying)
    // Use pivot for rotation/scaling from center if needed
    const waveY = Math.sin(time) * 10 * intensity;
    screen.y += waveY;

    // Working with filters
    if (intensity > 0) {
        // Create a new filter instance for the current frame
        // IMPORTANT: Do not modify screen.filters directly, use currentFrameFilters array
        const colorFilter = new PIXI.filters.ColorMatrixFilter();
        
        // Shift hue towards blue (180 degrees - cyan/blue)
        colorFilter.hue(180 * intensity, false);
        
        // Add filter to this frame's render stack
        currentFrameFilters.push(colorFilter);
        
        // Can add additional filters, e.g., Blur
        if (intensity > 0.5) {
             const blur = new PIXI.filters.BlurFilter();
             blur.blur = (intensity - 0.5) * 4;
             currentFrameFilters.push(blur);
        }
    }
};
```

---

## Parsing Nuances and Parameters

The system uses a built-in Notetag parser, which has specific behavior regarding arguments.

1.  **Standard Format:** `<DikarierEffectName>`
    *   `param`: `null`
    *   `fadeDuration`: Default value (180 frames).
2.  **Format with Argument:** `<DikarierEffectName: 60>`
    *   **Warning:** The parser automatically interprets the number after the colon as `fadeDuration` (in seconds), not as an arbitrary parameter.
    *   `param`: `null`
    *   `fadeDuration`: 60 seconds.
3.  **Exception (Slow):** `<DikarierEffectSlow: 0.5>`
    *   A special regex is written for this effect.
    *   `param`: `0.5`
    *   `fadeDuration`: Default value.

**To pass custom parameters to your effect:**
Do not rely on automatic notetag parsing if you need to pass a number other than fade time. Instead, use an API call:

```javascript
// Manually trigger effect with custom parameter
const durationSec = 10;
const fadeSec = 2;
const customParam = 42; // Your value

des.mainClass.applyEffect(
    '<DikarierEffectMyCustom>', 
    customParam, 
    fadeSec, 
    durationSec, 
    $gameParty.leader(), 
    null // stateId, if applicable
);
```

Inside `updateMyEffect`, you will be able to read `effectData.param` (equal to 42).

---

## Working with PIXI.js and Transforms

When working with the `screen` object (inherits from `PIXI.Container`), follow these rules to ensure compatibility and performance.

### 1. Transforms
Every frame, the plugin resets the main transformation parameters (`x`, `y`, `scale`, `rotation`, `skew`) before calling update functions. This means effects are **additive** within a single frame but do not accumulate between frames (stateless rendering).

*   **Correct:** `screen.x += 10;` (Shifts screen by 10 pixels in current frame).
*   **Useless:** `screen.x = screen.x + 1;` (Like a counter increment — will not work as `x` resets).

### 2. Filters
The system clears `screen.filters` every frame and rebuilds them based on the `pluginFilters` (aka `currentFrameFilters`) array.

*   **Always create new filter instances** or use an object pool.
*   **Performance:** `PIXI.filters.BlurFilter` and `ColorMatrixFilter` are relatively cheap. `NoiseFilter` or convolution filters can severely drop FPS on weak devices.

### 3. Critical Warning: Alpha Channel
It is **strictly not recommended** to use `ColorMatrixFilter` to change the alpha channel (transparency) of the entire container.

```javascript
// NEVER DO THIS IN A COLOR MATRIX
matrix[18] = 0.5; // Setting alpha
```

**Reason:** `screen` contains not only map tiles but also event sprites, the player, and sometimes parallax. Changing alpha via matrix will cause black/transparent pixels to "shine through" layers, or events will become semi-transparent, revealing the map background incorrectly. Use `brightness` or `tint` for darkening.

---

## Lifecycle Management

### `intensity`
This property is automatically calculated by the core.
*   **Fade In:** Grows from 0 to 1.
*   **Full:** Stays at 1.
*   **Fade Out:** Falls from 1 to 0.

Use `intensity` as a multiplier for all your visual manipulations.

```javascript
// Smoothly turn effect on/off
const shakePower = 5 * effectData.intensity; 
```

### Forced Stop
If you want to interrupt an effect prematurely:

```javascript
// Transitions effect to Fade Out stage
des.mainClass.clearEffect('<DikarierEffectMyCustom>'); 
```

---

## API Access (PRO features)

Although the API is open, functions for binding effects to maps (`MapEffect`) and equipment (`ItemEffect`) work through the `EffectUtils` class, which depends on initialization in `Dikarier_Core`. When developing addons, check for the existence of necessary methods if you are using these specific bindings.

For normal operation via States, no additional checks are required.
