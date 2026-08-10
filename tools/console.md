# console.html — Stereo Strip 仕様書

**対象ファイル**: `tools/console.html`
**タイトル**: Stereo Strip — Gate · Comp · EQ
**作成日**: 2026-08-10（既存実装からの後付け仕様書）

---

## 1. 概要

ミキシングコンソールのチャンネルストリップ（Gate → Compressor → EQ 4-band）を
ブラウザ単体で再現する教材ツール。単一 HTML ファイルで完結し、外部ライブラリ・
ビルド・サーバーを必要としない。

主な目的：

- Gate / Comp のパラメーターとゲインリダクションの関係を、耳とメーターの両方で体感する
- EQ 4-band のカーブと、PRE / POST のスペクトラムを重ねて確認する
- ヤマハ系デジタル卓に近いノブ・フェーダー操作感を再現する

音源は内蔵オシレーター（sawtooth）とローカル音声ファイルの 2 系統。

---

## 2. 動作環境

| 項目 | 内容 |
| ---- | ---- |
| 対応ブラウザ | AudioWorklet 対応の Chromium 系 / Safari / Firefox |
| 依存ライブラリ | なし（Vanilla JS + Canvas 2D + Web Audio API） |
| 実行方法 | HTML を直接開く。`file://` でも AudioWorklet は Blob URL 経由で動作 |
| サンプルレート | AudioContext を 48000 Hz 固定で生成 |
| 入力 | ブラウザ内で完結。ファイルはアップロードされない |

DSP は AudioWorklet で動作する。`addModule` が Blob URL・data URL の両方で失敗した
場合のみ ScriptProcessor（512 サンプル）にフォールバックし、ステータス欄に
`⚠ ScriptProcessor fallback` を表示する。

---

## 3. 画面構成

```
┌─ topbar ────────────────────────────────────────────┐
│ タイトル / ▶ OSC / ▶ File… / ■ Stop / ステータス      │
├─ filebar ───────────────────────────────────────────┤
│ ファイル名 / ↩ Loop / Rate ノブ                       │
├─ main (2 カラム) ───────────────────────────────────┤
│ 左カラム              │ 右カラム                     │
│  ・GATE               │  ・EQ 4-BAND                 │
│  ・COMPRESSOR         │    - PRE/POST FFT トグル      │
│  ・METERS             │    - EQ カーブ Canvas         │
│  ・I/O (Output fader) │    - 帯域別ノブ × 4           │
│                       │    - Smooth ノブ              │
└─────────────────────────────────────────────────────┘
```

セクション上端の色帯で機能を識別する：Gate = 緑、Comp = アンバー、
Meters = ブルー、EQ = パープル。

レスポンシブのブレークポイントは 1380 / 1280 / 1080 / 760 px。
1080px 以下で 1 カラムに畳まれる。

---

## 4. シグナルフロー

```
 [Oscillator] または [AudioBufferSource]
        │
        ├──────────────→ preAnalyser  （PRE FFT 表示用）
        │
        ▼
 [stereo-strip worklet]
   Gate → Comp → (Makeup × Output) → EQ 4-band
        │
        ▼
   postAnalyser （POST FFT 表示用）
        │
        ▼
   destination
```

- モノラルファイルは ChannelSplitter → ChannelMerger でステレオに複製してから入力する
- オシレーターは Gain 0.08 を通して L/R 両方に分配する
- メーター値は worklet から `postMessage` で 8 ブロックに 1 回送られる

---

## 5. DSP 仕様

### 5.1 検波（ディテクター）

Gate と Comp は共通の検波エンベロープ `detEnv` を使う。

- 入力: `x = max(|L|, |R|)`（ピーク検波）
- アタック 1 ms 固定、リリース 50 ms 固定の一次 IIR
- 係数: `alpha = exp(-1 / (ms × 0.001 × fs))`

### 5.2 Gate

| パラメーター | ID | 範囲 | 既定値 | 単位 |
| ---- | ---- | ---- | ---- | ---- |
| Threshold | `gTh` | -80 〜 0 | -50 | dB |
| Range | `gRange` | -80 〜 0 | -40 | dB |
| Attack | `gAtk` | 0.1 〜 100（対数） | 5 | ms |
| Hold | `gHold` | 0 〜 500 | 30 | ms |
| Release | `gRel` | 1 〜 1000（対数） | 120 | ms |
| ON/OFF | `gOn` | 0 / 1 | 1 | — |

動作：

1. `inDb >= gTh` ならターゲットゲイン 1.0、ホールドカウンタをリセット
2. しきい値を下回ってもホールド中はゲイン 1.0 を維持
3. ホールド終了後はターゲットを `Range`（線形換算値）まで落とす
4. ターゲットへはアタック / リリース係数で滑らかに追従する

`Range` が 0 dB のときゲートは実質無効、-80 dB でほぼ完全な無音になる。

### 5.3 Compressor

| パラメーター | ID | 範囲 | 既定値 | 単位 |
| ---- | ---- | ---- | ---- | ---- |
| Threshold | `cTh` | -60 〜 0 | -18 | dB |
| Ratio | `cRatio` | 1 〜 20 | 4 | :1 |
| Knee | `cKnee` | 0 〜 24 | 6 | dB |
| Attack | `cAtk` | 0.1 〜 100（対数） | 10 | ms |
| Release | `cRel` | 5 〜 2000（対数） | 150 | ms |
| Makeup | `cMake` | 0 〜 24 | 0 | dB |
| ON/OFF | `cOn` | 0 / 1 | 1 | — |

ゲインリダクション（dB）の算出：

- ハードニー（`Knee <= 0`）:
  `GR = -(inDb - Th) × (1 - 1/Ratio)`（`inDb > Th` のとき）
- ソフトニー（`Knee > 0`）: 膝の下端 `Th - Knee/2`、上端 `Th + Knee/2`
  - 膝より上: ハードニーと同じ式
  - 膝の中: `GR = -((1 - 1/Ratio) × (inDb - Th)²) / Knee`

`Ratio <= 1.0001` のときコンプは通過（ゲイン 1.0）となる。

### 5.4 EQ 4-band

すべて RBJ Audio EQ Cookbook の双二次フィルタ。直列 4 段。

| 帯域 | Freq 範囲 | 既定 Freq | Q 範囲 | 既定 Q | タイプ切替 |
| ---- | ---- | ---- | ---- | ---- | ---- |
| LOW | 20 〜 800 Hz | 120 Hz | 0.2 〜 10 | 0.707 | PEQ / HPF |
| LO-MID | 80 〜 4000 Hz | 400 Hz | 0.2 〜 10 | 1.000 | PEQ 固定 |
| HI-MID | 200 〜 12000 Hz | 2500 Hz | 0.2 〜 10 | 1.000 | PEQ 固定 |
| HIGH | 500 〜 20000 Hz | 8000 Hz | 0.2 〜 10 | 0.707 | PEQ / LPF |

Gain は全帯域 -18 〜 +18 dB、既定 0 dB。HPF / LPF 選択時は Gain を無視する。

- 中心周波数は `10 Hz 〜 fs × 0.49` にクランプされる
- Q は下限 0.2 にクランプされる
- 帯域 OFF、または EQ セクション OFF のときは係数をオールパス（b0=1、他 0）に置く

**係数スムージング**：ノブ操作時のジッパーノイズを避けるため、フィルタ係数
（b0, b1, b2, a1, a2）そのものを一次 IIR で補間する。時定数は Smooth ノブ
（`eqSmooth`、0 〜 100 ms、既定 20 ms）。0 ms のときは即時反映。

> 注：係数補間は簡便で軽いが、大きく係数が動く途中で一時的に不安定な極配置に
> なる可能性がある。この実装は教材用途としてその簡便さを優先している。

### 5.5 出力

`gTotal = gainGate × gainComp × makeupLin × outLin` を EQ の前段で乗算する。
つまり EQ は Output フェーダーより後ろに位置する。

---

## 6. UI 仕様

### 6.1 ノブ（`.knob-canvas`）

Canvas 2D で描画するロータリーノブ。パラメーターは `data-*` 属性で宣言する。

| 属性 | 意味 |
| ---- | ---- |
| `data-id` | `PARAMS` のキー名 |
| `data-min` / `data-max` | 値域 |
| `data-val` | 既定値（ダブルクリックで復帰する値） |
| `data-step` | スナップ幅 |
| `data-fmt` | 表示書式（`0f` / `1f` / `2f` / `3f`） |
| `data-log` | `1` で対数スケール |

操作：

| 操作 | 動作 |
| ---- | ---- |
| 上下ドラッグ | 値の増減（感度 0.008 / 正規化値・px） |
| Shift + ドラッグ | 微調整（感度 0.002） |
| ホイール | 値の増減（0.004、Shift で 0.001） |
| ダブルクリック | `data-val` の既定値に戻す |

見た目：135°〜405°（掃引 270°）の指示範囲、金属リング、値アークとポインタ線。
EQ の Freq / Gain ノブは帯域色（LOW 水色 / LO-MID 緑 / HI-MID 黄 / HIGH 桃）で
着色し、それ以外はブルーのアクセントを使う。DPR に応じて解像度を上げる。

### 6.2 EQ カーブ Canvas

- 横軸：20 Hz 〜 20 kHz の対数、グリッド線は 20/50/100/200/500/1k/2k/5k/10k/20k
- 縦軸：EQ ゲイン ±18 dB（グリッドは ±12 / ±6 / 0 dB）
- 表示要素：
  - 各帯域のカーブ（帯域色、OFF 時は減光）
  - 合成カーブ（白・太線）
  - 帯域ハンドル（L / LM / HM / H のラベル付き円）
  - PRE FFT（青）/ POST FFT（オレンジ）のスペクトラム
- スペクトラムの縦軸は 0 〜 -96 dBFS で、右端に `0 dBFS` / `-48` / `-96` を表示する
  （EQ ゲイン軸とスペクトラム軸は別スケールで重ねている）
- カーブ計算は `EQ_SAMPLE_RATE = 48000` 固定

カーブの計算（`bandMag()`）は 2 段階に分かれる。

1. 帯域の**中心周波数** `f0` で双二次係数を設計する（§5.4 と同じ RBJ の式。
   worklet が実際に使う係数と一致する）
2. 描画する各周波数 `freqHz` で `|H(e^{jw})|` を評価する

この 2 つを混同して評価周波数で係数を設計すると `|H|` が定数に潰れる。
§8.2 に記録した不具合がこれだった。

操作：

| 操作 | 動作 |
| ---- | ---- |
| ハンドルを左ドラッグ | 横 = Freq、縦 = Gain |
| ハンドルを右ドラッグ | Q（上で増加） |
| ハンドル上でホイール | Q |

ドラッグで変更した値は対応するノブの表示・描画に同期する。
ヒット判定はハンドル中心から 12 px 以内。

### 6.3 メーター

縦型メーター 4 本。worklet からの `postMessage` で更新する。

| メーター | 表示 | レンジ |
| ---- | ---- | ---- |
| Input | 検波後ピーク | -60 〜 0 dB（下から伸びる） |
| Gate GR | ゲートの GR 最小値 | 0 〜 -40 dB（上から伸びる） |
| Comp GR | コンプの GR 最小値 | 0 〜 -40 dB（上から伸びる） |
| Total GR | Gate + Comp の合計 GR | 0 〜 -40 dB（上から伸びる） |

数値表示は `-inf`（-200 dB 未満）と符号付き小数第 1 位。

### 6.4 Output フェーダー

卓のマスターフェーダー風。範囲 -24 〜 +12 dB、既定 0 dB、スナップ 0.1 dB。
トラックまたはキャップのドラッグで操作、ダブルクリックで 0.0 dB に復帰する。

### 6.5 トランスポート

| 操作 | 動作 |
| ---- | ---- |
| ▶ OSC | 内蔵オシレーター（sawtooth、Osc Hz ノブ 50 〜 2000 Hz、既定 220 Hz）を再生 |
| ▶ File… | ローカル音声ファイルを選択して再生 |
| ■ Stop | 全ノードを停止・切断し AudioContext を close |
| ↩ Loop | ファイル再生のループ切替（再生中も反映） |
| Rate | 再生速度 0.5 〜 2.0 倍（既定 1.00） |

再生開始時は必ず `stopAll()` を通してから新しい AudioContext を作り直す。

---

## 7. 内部構造

| 区分 | 内容 |
| ---- | ---- |
| `workletCode` | AudioWorkletProcessor `stereo-strip` のソースを文字列で保持 |
| `createScriptProcessorStrip()` | worklet と同一アルゴリズムの ScriptProcessor 版フォールバック |
| `PARAMS` | 全パラメーターの単一の真実の源（plain object） |
| `UISTATE` | FFT 表示など UI だけの状態 |
| `pushParams()` | `PARAMS` を AudioParam へ一括反映 |
| `onParamChange()` | パラメーター変更時の共通フック（反映 + 再描画） |

`PARAMS` の値域と worklet の `parameterDescriptors` の値域は意図的に一致していない。
worklet 側を広め（例：Gain ±24 dB、Ratio 上限 40）に取り、UI 側で実用域に絞っている。

---

## 8. 不具合と注意点

### 8.1 初期化が途中で停止する（修正済み・2026-08-10）

`preAnalyser` / `postAnalyser` / `fftBuffer` / `vizRaf` の 4 つの変数が
どこにも宣言されていなかった。`drawSpectrumOverlay()` の先頭が未宣言の
`preAnalyser` を読むため ReferenceError が発生していた。

この関数は初期化時の `initEqCanvas()` → `resizeEq()` → `drawEqCanvas()` から
必ず呼ばれるため、スクリプトは初期化の途中（`initEqCanvas()` の行）で中断していた。
スクリプト全体が `(async () => {...})()` で囲まれているため、エラーは未処理の
Promise rejection になり、コンソールに何も出ないまま静かに失敗していた。

修正前にローカル HTTP サーバー上で確認した症状：

- EQ カーブ Canvas にグリッドしか描かれない（帯域カーブ・合成カーブ・ハンドルなし）
- ▶ OSC / ▶ File… / ■ Stop が反応しない（リスナー未登録）
- Gate / Comp / EQ の ON/OFF ボタン、タイプ切替が反応しない
- Output フェーダーのキャップが表示されず操作できない
- ノブは描画・ドラッグできるが、値を変えても EQ カーブが再描画されない

**修正内容**：`ctx_audio` などの宣言と同じ場所（`console.html:2054`）に以下を追加した。

```js
let preAnalyser = null, postAnalyser = null;
let fftBuffer = null, vizRaf = 0;
```

これらの宣言はトップレベルの文として `init` 呼び出しより前に評価されるため、
`let` の TDZ には触れない。

**修正後の動作確認**（ローカル HTTP サーバー + Chromium）：

- EQ カーブ、合成カーブ、L / LM / HM / H ハンドルが描画される
- ▶ OSC で `running (osc) @ 48000 Hz` になり、AudioWorklet 経路で動作する
  （ScriptProcessor へのフォールバックなし）
- 220 Hz ノコギリ波の倍音列が PRE / POST 両方のスペクトラムに表示される
- Input メーターが振れる（-24.4 dB / バー 59.3%）
- Comp Threshold を -60 dB まで下げると Comp GR / Total GR が -26.7 dB を示し、
  POST スペクトラムが PRE より下がる
- HI-MID の Gain ノブをドラッグすると EQ カーブとハンドルが追従する
- ■ Stop で `stopped` に戻る

### 8.2 EQ カーブが Freq と Q を反映しない（修正済み・2026-08-10）

`bandMag()` が双二次係数を**評価周波数**から設計していた。中心周波数 `f0` は
計算されていたが、その後どこでも使われていなかった。

```js
const w  = 2*Math.PI*freqHz/fs;             // 描画する周波数
const cosw=Math.cos(w), sinw=Math.sin(w), alpha=sinw/(2*Q);
b0=1+alpha*A; b1=-2*cosw; b2=1-alpha*A;     // ← f0 ではなく w で設計している
```

係数と評価点が同じ `w` になると、分子・分母が約分されて `|H|` が周波数に依らない
定数に潰れる。

- PEQ：`|H| = A² = 10^(G/20)`（全帯域でゲインぶんの水平線）
- HPF / LPF：`|H| = Q`（同じく水平線）

このため、カーブは常に水平線として描かれ、Q も中心周波数も形に現れなかった。
ハンドルの位置だけが Freq / Gain に追従するので、一見それらしく見えていた。

実測：LO-MID の Q ノブを 0.200 → 10.000 に振っても、Canvas のピクセルが
1 バイトも変化しなかった。

**修正内容**：`f0` で係数を設計し、`freqHz` で `|H(e^{jw})|` を評価する 2 段階に
分けた（`console.html:1646` 付近）。設計式は worklet 側の `biquadPeaking` /
`biquadHighpass` / `biquadLowpass` と一致するため、描画カーブが実際の処理と
合うようになった。未使用だった `re_b` / `im_b` も削除した。

**修正後の検証**：描画されたカーブのピクセルを読み取り、独立に実装した
RBJ の応答計算と突き合わせた。

- PEQ 400 Hz / +13 dB / Q=1：56 Hz 〜 5 kHz の 8 点で誤差 ±0.2 dB 以内
  （残差は線幅 2 px ぶんの量子化）
- 同じ帯域で Q=10 にしたとき、カーブの -3 dB 帯域幅から逆算した Q は 9.73
- HPF 120 Hz / Q=0.707：60 Hz で -12.09 dB（理論値 -12.31 dB、12 dB/oct）、
  240 Hz 以上はほぼフラット

### 8.3 その他の注意点

- ノブとフェーダーは `mousedown` / `mousemove` のみを見ており、
  `pointerdown` を使っていないためタッチ操作に対応していない
- `drawEqCanvas()` を `requestAnimationFrame` で毎フレーム呼ぶため、
  帯域カーブを毎回 Canvas 幅ぶんのサンプル数で再計算している（負荷が高い）
- スペクトラムの dB 軸と EQ ゲイン軸が別スケールのまま重なっており、
  カーブとスペクトラムの高さを直接比較することはできない

---

## 9. 変更履歴

| 版 | 日付 | 内容 |
| ---- | ---- | ---- |
| v3.1 | — | ヤマハ卓風レイアウトへ刷新（CSS 内コメントより） |
| — | 2026-08-10 | 本仕様書を既存実装から起こす |
| — | 2026-08-10 | §8.1 の初期化中断バグを修正（Analyser 系変数の宣言追加） |
| — | 2026-08-10 | §8.2 の EQ カーブ計算バグを修正（係数を中心周波数で設計するよう変更） |
