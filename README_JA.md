# Dikarier Effect System API (v2.6)

[Read in English](README_EN.md) | [На русском](README_RU.md) | [中文阅读](README_ZH.md)

## はじめに

このドキュメントは、独自の視覚効果（アドオン）を作成したり、**Dikarier Effect System** の機能を拡張したい開発者向けです。

このプラグインは、`PIXI.Container`（ゲーム画面）を操作し、ステート（State）やイベントに基づいてポストプロセスフィルタを適用するためのグローバルインターフェースを提供します。

> **開発者向けライセンス契約:**
> このAPIを使用して独自のアドオンや拡張機能を作成、配布、販売する権利を有します。
> プラグイン本体 `Dikarier_EffectSystem.js` のソースコードの再販や再配布は禁止されています。
> 視覚効果を動作させるには、ユーザーのプロジェクトに **Dikarier_Core** がインストールされている必要があります。

### ⚠️ 重要: 命名規則

プラグインのパーサーは、パフォーマンスを最適化するために厳格なタグフィルタリングを使用しています。
システムがエフェクトを認識し、処理ループに追加するためには、その名前（タグ）は**必ず**以下の形式に従う必要があります。

` <DikarierEffectName> `

*   プレフィックス `DikarierEffect` は必須です。
*   名前は CamelCase（キャメルケース）を推奨します。
*   タグは `>` で終了する必要があります。

**例:**
*   ✅ `<DikarierEffectRadiation>` — 正しい形式です。処理されます。
*   ❌ `<RadiationEffect>` — 無視されます（プレフィックスがありません）。
*   ❌ `<DikarierRadiation>` — 無視されます（プレフィックスが不完全です）。
*   ❌ `<MyCustomTag>` — 無視されます。

#### 関数の自動バインディング
エフェクトを処理する際、システムはタグから派生した名前を持つ更新関数を自動的に検索します。
タグ `<DikarierEffectFire>` の場合、システムは `updateFireEffect` という関数を探します。

---

## アーキテクチャとグローバルアクセス

すべての機能はグローバル名前空間 `des` を介して利用可能です。メインのエフェクト管理クラスは `des.mainClass` にあります。

```javascript
// エフェクトシステムのメインクラス
const EffectSystem = des.mainClass;

// 登録されたエフェクトのレジストリ
const Registry = des.mainClass.effectRegistry;

// 現在のフレームでアクティブなエフェクトの配列
const ActiveEffects = des.mainClass.activeEffects;
```

システムは `Scene_Base` のレンダリングに割り込み、フレームが描画される前に `Spriteset_Map` または `Spriteset_Battle` のプロパティを変更することで動作します。

---

## `effectData` オブジェクトの構造

各アクティブエフェクトは `effectData` オブジェクトとして表されます。このオブジェクトは毎フレーム、エフェクトの更新関数に渡されます。エフェクトの状態を管理するためには、この構造を理解することが重要です。

| プロパティ | 型 | 説明 |
| :--- | :--- | :--- |
| `id` | `String` | エフェクトインスタンスの一意のID（自動生成）。 |
| `effect` | `String` | エフェクトの完全なタグ名（例: `<DikarierEffectMyCustom>`）。 |
| `param` | `Mixed` | 追加パラメータ。**重要:** 標準のタグパーサー `<Tag: X>` を使用する場合、値 `X` は `fadeDuration`（フェード時間）として解釈され、`param` は `null` のままです（Slowエフェクトを除く）。カスタムパラメータを渡すには、手動で `applyEffect` を呼び出す必要があります。 |
| `actor` | `Game_Actor` | エフェクトが適用されているアクター。 |
| `stateId` | `Number` | データベースのステートID（ステートによってエフェクトが呼び出された場合）。 |
| `totalDuration` | `Number` | エフェクトの総持続時間（フレーム数）。ステートに依存する場合は `Infinity`。 |
| `fadeDuration` | `Number` | フェードインおよびフェードアウトの持続時間（フレーム数）。 |
| `elapsed` | `Number` | エフェクト開始から経過したフレーム数。 |
| `intensity` | `Number` | 現在のエフェクト強度（`0.0` から `1.0`）。システムによって自動計算されます。 |
| `state` | `String` | 現在のフェーズ: `'fadeIn'`, `'full'`, `'fadeOut'`。 |

---

## エフェクトの作成と登録

新しいエフェクトを追加するには、`effectRegistry` に更新関数を登録する必要があります。この関数は、アクティブなエフェクトに対して毎フレーム呼び出されます。

### 更新関数のシグネチャ

```javascript
/**
 * @param {Object} screen - PIXI.Container インスタンス（通常は Spriteset）。
 * @param {Object} effectData - 現在のエフェクトのデータオブジェクト。
 * @param {Array} currentFrameFilters - 現在のフレームの PIXI フィルタ配列。
 */
function updateMyEffect(screen, effectData, currentFrameFilters) {
    // エフェクトのロジック
}
```

### 実装例

以下は、「Underwater（水中）」エフェクトを作成する例です。波打つ歪みと青い色調を追加します。

```javascript
// エフェクトタグの登録
des.mainClass.effectRegistry['<DikarierEffectUnderwater>'] = function(screen) {
    // 初回実行時にコンテナ固有のプロパティを初期化
    screen._underwaterTime = screen._underwaterTime || { time: 0 };
};

// 更新ロジックの追加
// 関数名は次のパターンに従う必要があります: update + エフェクト名 + Effect
des.mainClass.updateUnderwaterEffect = function(screen, effectData, currentFrameFilters) {
    // アニメーション同期のためのグローバルエフェクト時間を取得
    const time = des.mainClass.effectTime * 0.05;
    
    // 計算された強度を取得（フェードイン/アウトを考慮）
    const intensity = effectData.intensity;

    // 画面の変形を適用（揺れ）
    // 必要に応じて pivot を使用して中心から回転/拡大縮小を行う
    const waveY = Math.sin(time) * 10 * intensity;
    screen.y += waveY;

    // フィルタの操作
    if (intensity > 0) {
        // 現在のフレーム用の新しいフィルタインスタンスを作成
        // 重要: screen.filters を直接変更せず、currentFrameFilters 配列を使用してください
        const colorFilter = new PIXI.filters.ColorMatrixFilter();
        
        // 色相を青方向にシフト（180度 - シアン/青）
        colorFilter.hue(180 * intensity, false);
        
        // このフレームのレンダリングスタックにフィルタを追加
        currentFrameFilters.push(colorFilter);
        
        // 追加のフィルタ（例：ブラー）を加えることも可能
        if (intensity > 0.5) {
             const blur = new PIXI.filters.BlurFilter();
             blur.blur = (intensity - 0.5) * 4;
             currentFrameFilters.push(blur);
        }
    }
};
```

---

## パーシングの注意点とパラメータ

システムは組み込みのノートタグ（Notetag）パーサーを使用しており、引数の処理に関して特定の動作をします。

1.  **標準フォーマット:** `<DikarierEffectName>`
    *   `param`: `null`
    *   `fadeDuration`: デフォルト値（180フレーム）。
2.  **引数付きフォーマット:** `<DikarierEffectName: 60>`
    *   **注意:** パーサーはコロンの後の数値を任意のパラメータではなく、自動的に `fadeDuration`（秒単位）として解釈します。
    *   `param`: `null`
    *   `fadeDuration`: 60秒。
3.  **例外 (Slow):** `<DikarierEffectSlow: 0.5>`
    *   このエフェクトには特別な正規表現が記述されています。
    *   `param`: `0.5`
    *   `fadeDuration`: デフォルト値。

**カスタムパラメータをエフェクトに渡す場合:**
フェード時間以外の数値を渡す必要がある場合は、自動ノートタグ解析に頼らないでください。代わりにAPI呼び出しを使用します。

```javascript
// カスタムパラメータを指定してエフェクトを手動で開始
const durationSec = 10;
const fadeSec = 2;
const customParam = 42; // あなたの値

des.mainClass.applyEffect(
    '<DikarierEffectMyCustom>', 
    customParam, 
    fadeSec, 
    durationSec, 
    $gameParty.leader(), 
    null // 該当する場合は stateId
);
```

`updateMyEffect` 内で、`effectData.param`（42になる）を読み取ることができます。

---

## PIXI.js とトランスフォーム（変形）の扱い

`screen` オブジェクト（`PIXI.Container` を継承）を扱う際は、互換性とパフォーマンスを確保するために以下のルールに従ってください。

### 1. トランスフォーム (Transform)
各フレームで、プラグインは更新関数を呼び出す前に、主要な変形パラメータ（`x`, `y`, `scale`, `rotation`, `skew`）をリセットします。これは、エフェクトが同一フレーム内では**加算的**ですが、フレーム間では蓄積されない（ステートレスレンダリング）ことを意味します。

*   **正しい:** `screen.x += 10;` (現在のフレームで画面を10ピクセルずらす)。
*   **無意味:** `screen.x = screen.x + 1;` (カウンタのインクリメントのような処理 — `x` がリセットされるため動作しません)。

### 2. フィルタ (Filters)
システムは毎フレーム `screen.filters` をクリアし、`pluginFilters`（または `currentFrameFilters`）配列に基づいて再構築します。

*   **常に新しいフィルタインスタンスを作成**するか、オブジェクトプールを使用してください。
*   **パフォーマンス:** `PIXI.filters.BlurFilter` や `ColorMatrixFilter` は比較的軽量です。`NoiseFilter` や畳み込み（Convolution）フィルタは、低スペックなデバイスでFPSを著しく低下させる可能性があります。

### 3. 重大な警告: アルファチャンネル
`ColorMatrixFilter` を使用してコンテナ全体のアルファチャンネル（透明度）`alpha` を変更することは**強く推奨されません**。

```javascript
// カラーマトリックスでこれを行わないでください
matrix[18] = 0.5; // アルファの設定
```

**理由:** `screen` にはマップタイルだけでなく、イベントスプライト、プレイヤー、場合によってはパララックスも含まれます。マトリックス経由でアルファを変更すると、黒や透明なピクセルがレイヤーを「透過」したり、イベントが半透明になってマップの背景が不適切に見えたりする原因になります。暗くするには `brightness`（明度）や `tint`（色合い）を使用してください。

---

## ライフサイクル管理

### `intensity` (強度)
このプロパティはコアによって自動的に計算されます。
*   **Fade In:** 0 から 1 へ増加。
*   **Full:** 1 を維持。
*   **Fade Out:** 1 から 0 へ減少。

`intensity` をすべての視覚操作の乗数として使用してください。

```javascript
// エフェクトの滑らかなオン/オフ
const shakePower = 5 * effectData.intensity; 
```

### 強制停止
エフェクトを途中で中断したい場合:

```javascript
// エフェクトを Fade Out ステージへ移行させる
des.mainClass.clearEffect('<DikarierEffectMyCustom>'); 
```

---

## API アクセス (PRO機能)

APIは公開されていますが、マップ (`MapEffect`) や装備 (`ItemEffect`) にエフェクトをバインドする機能は、`Dikarier_Core` の初期化に依存する `EffectUtils` クラスを通じて動作します。アドオンを開発する際、これらの特定のバインドを使用する場合は、必要なメソッドが存在するか確認してください。

ステート (States) を介した通常の動作には、追加のチェックは不要です。