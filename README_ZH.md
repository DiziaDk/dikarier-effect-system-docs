# Dikarier Effect System API (v2.6)

[Read in English](README_EN.md) | [日本語で読む](README_JA.md) | [На русском](README_RU.md)

## 简介

本文档旨在帮助开发者创建自定义视觉效果（插件/Addons）或扩展 **Dikarier Effect System** 的功能。

该插件提供了一个全局接口，用于操作 `PIXI.Container`（游戏屏幕）并根据状态（State）或事件应用后期处理滤镜。

> **开发者许可协议：**
> 您有权创建、分发和销售使用此 API 的自定义插件/扩展。
> 禁止转售或分发 `Dikarier_EffectSystem.js` 插件本身的源代码。
> 用户项目中必须安装 **Dikarier_Core** 才能使视觉效果正常工作。

### ⚠️ 关键：命名规范

插件的解析器使用严格的标签过滤来优化性能。
为了让系统识别您的效果并将其添加到处理循环中，其名称（标签）**必须**遵循以下格式：

` <DikarierEffectName> `

*   前缀 `DikarierEffect` 是必须的。
*   名称建议使用驼峰命名法（CamelCase）。
*   标签必须以 `>` 结尾。

**示例：**
*   ✅ `<DikarierEffectRadiation>` — 正确。将被处理。
*   ❌ `<RadiationEffect>` — 忽略（无前缀）。
*   ❌ `<DikarierRadiation>` — 忽略（前缀不完整）。
*   ❌ `<MyCustomTag>` — 忽略。

#### 自动函数绑定
在处理效果时，系统会根据标签派生的名称自动搜索更新函数。
对于标签 `<DikarierEffectFire>`，系统将查找函数 `updateFireEffect`。

---

## 架构与全局访问

所有功能都可以通过全局命名空间 `des` 访问。主要的效果管理类位于 `des.mainClass` 中。

```javascript
// 效果系统主类
const EffectSystem = des.mainClass;

// 已注册效果的注册表
const Registry = des.mainClass.effectRegistry;

// 当前帧中的活动效果数组
const ActiveEffects = des.mainClass.activeEffects;
```

该系统通过拦截 `Scene_Base` 的渲染并在绘制帧之前修改 `Spriteset_Map` 或 `Spriteset_Battle` 的属性来工作。

---

## `effectData` 对象结构

每个活动效果都由一个 `effectData` 对象表示。该对象每帧都会传递给效果的更新函数。理解此结构对于管理效果状态至关重要。

| 属性 | 类型 | 描述 |
| :--- | :--- | :--- |
| `id` | `String` | 效果实例的唯一标识符（自动生成）。 |
| `effect` | `String` | 效果的完整标签名（例如 `<DikarierEffectMyCustom>`）。 |
| `param` | `Mixed` | 附加参数。**重要：** 使用标准标签解析器 `<Tag: X>` 时，值 `X` 被解释为 `fadeDuration`（淡入淡出时间），而 `param` 保持为 `null`（Slow 效果除外）。传递自定义参数需要手动调用 `applyEffect`。 |
| `actor` | `Game_Actor` | 被施加效果的角色。 |
| `stateId` | `Number` | 数据库中的状态 ID（如果效果由状态触发）。 |
| `totalDuration` | `Number` | 效果的总持续时间（帧数）。如果持续时间取决于状态，则为 `Infinity`。 |
| `fadeDuration` | `Number` | 淡入（Fade In）和淡出（Fade Out）阶段的持续时间（帧数）。 |
| `elapsed` | `Number` | 自效果开始以来经过的帧数。 |
| `intensity` | `Number` | 当前效果强度（从 `0.0` 到 `1.0`）。由系统自动计算。 |
| `state` | `String` | 当前阶段：`'fadeIn'`、`'full'` 或 `'fadeOut'`。 |

---

## 创建和注册效果

要添加新效果，必须在 `effectRegistry` 中注册一个更新函数。该函数会对活动效果每帧调用一次。

### 更新函数签名

```javascript
/**
 * @param {Object} screen - PIXI.Container 实例（通常是 Spriteset）。
 * @param {Object} effectData - 当前效果的数据对象。
 * @param {Array} currentFrameFilters - 当前帧的 PIXI 滤镜数组。
 */
function updateMyEffect(screen, effectData, currentFrameFilters) {
    // 效果逻辑
}
```

### 实现示例

下面是一个创建“水下（Underwater）”效果的示例，它添加了波浪扭曲和蓝色色调。

```javascript
// 注册效果标签
des.mainClass.effectRegistry['<DikarierEffectUnderwater>'] = function(screen) {
    // 首次运行时初始化容器的唯一属性
    screen._underwaterTime = screen._underwaterTime || { time: 0 };
};

// 添加更新逻辑。
// 函数名必须遵循模式：update + 效果名 + Effect
des.mainClass.updateUnderwaterEffect = function(screen, effectData, currentFrameFilters) {
    // 获取全局效果时间以同步动画
    const time = des.mainClass.effectTime * 0.05;
    
    // 获取计算出的强度（考虑淡入/淡出）
    const intensity = effectData.intensity;

    // 应用屏幕变换（摇摆）
    // 如果需要，使用 pivot 从中心旋转/缩放
    const waveY = Math.sin(time) * 10 * intensity;
    screen.y += waveY;

    // 处理滤镜
    if (intensity > 0) {
        // 为当前帧创建一个新的滤镜实例
        // 重要：不要直接修改 screen.filters，请使用 currentFrameFilters 数组
        const colorFilter = new PIXI.filters.ColorMatrixFilter();
        
        // 将色相向蓝色偏移（180度 - 青色/蓝色）
        colorFilter.hue(180 * intensity, false);
        
        // 将滤镜添加到此帧的渲染堆栈中
        currentFrameFilters.push(colorFilter);
        
        // 可以添加额外的滤镜，例如模糊（Blur）
        if (intensity > 0.5) {
             const blur = new PIXI.filters.BlurFilter();
             blur.blur = (intensity - 0.5) * 4;
             currentFrameFilters.push(blur);
        }
    }
};
```

---

## 解析细节和参数

系统使用内置的备注栏（Notetag）解析器，该解析器在处理参数时具有特定的行为。

1.  **标准格式：** `<DikarierEffectName>`
    *   `param`: `null`
    *   `fadeDuration`: 默认值（180帧）。
2.  **带参数的格式：** `<DikarierEffectName: 60>`
    *   **注意：** 解析器会自动将冒号后的数字解释为 `fadeDuration`（以秒为单位），而不是任意参数。
    *   `param`: `null`
    *   `fadeDuration`: 60 秒。
3.  **例外 (Slow)：** `<DikarierEffectSlow: 0.5>`
    *   为此效果编写了特殊的正则表达式。
    *   `param`: `0.5`
    *   `fadeDuration`: 默认值。

**要将自定义参数传递给您的效果：**
如果您需要传递除淡入淡出时间以外的数字，请勿依赖自动备注栏解析。相反，请使用 API 调用：

```javascript
// 使用自定义参数手动触发效果
const durationSec = 10;
const fadeSec = 2;
const customParam = 42; // 您的值

des.mainClass.applyEffect(
    '<DikarierEffectMyCustom>', 
    customParam, 
    fadeSec, 
    durationSec, 
    $gameParty.leader(), 
    null // stateId，如果适用
);
```

在 `updateMyEffect` 内部，您将能够读取 `effectData.param`（等于 42）。

---

## 使用 PIXI.js 和变换（Transforms）

在处理 `screen` 对象（继承自 `PIXI.Container`）时，请遵循以下规则以确保兼容性和性能。

### 1. 变换（Transform）
在每一帧，插件都会在调用更新函数之前重置主要变换参数（`x`, `y`, `scale`, `rotation`, `skew`）。这意味着效果在单帧内是**叠加**的，但不会在帧之间累积（无状态渲染）。

*   **正确：** `screen.x += 10;` （在当前帧将屏幕移动 10 像素）。
*   **无用：** `screen.x = screen.x + 1;` （像计数器增量一样——不起作用，因为 `x` 会被重置）。

### 2. 滤镜（Filters）
系统每帧都会清除 `screen.filters` 并根据 `pluginFilters`（即 `currentFrameFilters`）数组重新构建它们。

*   **始终创建新的滤镜实例**或使用对象池。
*   **性能：** `PIXI.filters.BlurFilter` 和 `ColorMatrixFilter` 相对便宜。`NoiseFilter` 或卷积滤镜可能会严重降低低端设备的 FPS。

### 3. 严重警告：Alpha 通道
**强烈不建议**使用 `ColorMatrixFilter` 更改整个容器的 Alpha 通道（透明度）。

```javascript
// 切勿在颜色矩阵中这样做
matrix[18] = 0.5; // 设置 Alpha
```

**原因：** `screen` 不仅包含地图图块，还包含事件精灵、玩家以及有时包含远景图。通过矩阵更改 Alpha 会导致黑色/透明像素“穿透”图层，或者事件变得半透明，从而错误地显示地图背景。请使用 `brightness`（亮度）或 `tint`（色调）变暗。

---

## 生命周期管理

### `intensity` (强度)
此属性由核心自动计算。
*   **Fade In (淡入):** 从 0 增长到 1。
*   **Full (全效):** 保持为 1。
*   **Fade Out (淡出):** 从 1 下降到 0。

使用 `intensity` 作为所有视觉操作的乘数。

```javascript
// 平滑开启/关闭效果
const shakePower = 5 * effectData.intensity; 
```

### 强制停止
如果您想提前中断效果：

```javascript
// 将效果转换为 Fade Out 阶段
des.mainClass.clearEffect('<DikarierEffectMyCustom>'); 
```

---

## API 访问（PRO 功能）

虽然 API 是开放的，但将效果绑定到地图 (`MapEffect`) 和装备 (`ItemEffect`) 的功能是通过 `EffectUtils` 类工作的，该类依赖于 `Dikarier_Core` 中的初始化。开发插件时，如果您使用这些特定绑定，请检查是否存在必要的方法。

对于通过状态 (States) 的常规操作，不需要额外的检查。