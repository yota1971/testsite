# macOS + ffmpeg + Automator  
## ラウドネスノーマライズ自動化

---

## 1. 概要

この構成で実現すること：

- フォルダに入れるだけで自動処理（Automator）
- ラウドネス統一（EBU R128 / 2パス）
- 出力フォーマット統一（例：FLAC）
- 24bit相当の精度維持
- サンプルレート自動維持（44.1kHz → 44.1kHz など）
- メタデータ保持
- 日本語ファイル名・スペース対応


---

## 2. 必要環境の構築

### 2.1 Homebrewをインストール
terminalで

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```
### Apple Siliconでのインストール状態確認

```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```
### 2.2 必要ツールをインストール

```bash
brew install ffmpeg jq
```
* ffmpeg：音声処理用ライブラリ
* jq：JSONパース用ライブラリ

## 3. 入出力フォルダを構成

```bash
mkdir -p ~/Audio/{in,out,processed,log}
```

| フォルダ | 用途 |
| ---- | ---- |
| in | 入力 |
| out | 出力 | 
|processed|処理済み|
|log|ログ|

## 4. Automatorの設定
1. Automatorを起動
2. 「フォルダアクション」作成
3. 対象フォルダ：~/Audio/in
4. 「シェルスクリプトを実行」追加
* シェル：/bin/bash
* 入力：引数として渡す
5. 下記のスクリプトをペースト

```bash
#!/bin/bash

# =========================================================
# 変換ポリシー（ここだけ変更すれば挙動が変わる）
# =========================================================

# 出力ファイルのフォーマットを指定
OUTPUT_FORMAT="flac"   # flac / wav / aac / mp3

# 出力ファイルのLUFSを指定 デフォルトは-16LUFS
TARGET_I="-16"
TARGET_TP="-1.5"
TARGET_LRA="11"

# 出力ファイルのサンプルレートを指定 autoなら入力ファイルと同じ
SAMPLE_RATE_MODE="auto"   # auto / fixed
# ↑がfixedの場合ここで指定したfsになる
FORCE_SAMPLE_RATE="44100"

# 出力ファイルのbit depth
BIT_DEPTH="24"            # 16 / 24 / 32

# アートワークを残すかどうか
KEEP_ARTWORK="false"      # true / false

# =========================================================
# ここから下は編集非推奨
# =========================================================

# =========================================================
# 実行バイナリ
# =========================================================

FFMPEG="/opt/homebrew/bin/ffmpeg"
FFPROBE="/opt/homebrew/bin/ffprobe"
JQ="/opt/homebrew/bin/jq"

# =========================================================
# ディレクトリ
# =========================================================

BASE="$HOME/Audio"
OUT_DIR="$BASE/out"
PROCESSED_DIR="$BASE/processed"
LOG_DIR="$BASE/log"
LOG_FILE="$LOG_DIR/process.log"

mkdir -p "$OUT_DIR" "$PROCESSED_DIR" "$LOG_DIR"

echo "==== $(date) START ====" >> "$LOG_FILE"

# =========================================================
# ビットデプス
# =========================================================

case "$BIT_DEPTH" in
  16) SAMPLE_FMT="s16" ;;
  24) SAMPLE_FMT="s32" ;;  # 実質24bit
  32) SAMPLE_FMT="s32" ;;
  *) SAMPLE_FMT="s32" ;;
esac

# =========================================================
# フォーマット
# =========================================================

case "$OUTPUT_FORMAT" in
  flac)
    CODEC_BASE="-c:a flac -sample_fmt $SAMPLE_FMT"
    EXT="flac"
    ;;
  wav)
    CODEC_BASE="-c:a pcm_${SAMPLE_FMT}le"
    EXT="wav"
    ;;
  aac)
    CODEC_BASE="-c:a aac -b:a 192k"
    EXT="m4a"
    ;;
  mp3)
    CODEC_BASE="-c:a libmp3lame -b:a 192k"
    EXT="mp3"
    ;;
  *)
    CODEC_BASE="-c:a flac -sample_fmt s32"
    EXT="flac"
    ;;
esac

# =========================================================
# メイン処理
# =========================================================

for f in "$@"; do

  [ ! -f "$f" ] && continue

  # コピー完了待ち
  prev=-1
  while true; do
    size=$(stat -f%z "$f")
    [ "$size" -eq "$prev" ] && break
    prev=$size
    sleep 2
  done

  filename=$(basename "$f")
  name="${filename%.*}"
  OUT_FILE="$OUT_DIR/${name}.${EXT}"

  # サンプルレート取得
  INPUT_SR=$($FFPROBE -v error \
    -select_streams a:0 \
    -show_entries stream=sample_rate \
    -of default=noprint_wrappers=1:nokey=1 \
    "$f")

  # サンプルレート決定
  if [ "$SAMPLE_RATE_MODE" = "auto" ]; then
    OUTPUT_SR="$INPUT_SR"
  else
    OUTPUT_SR="$FORCE_SAMPLE_RATE"
  fi

  # マッピング
  if [ "$KEEP_ARTWORK" = "true" ]; then
    MAP_OPT="-map 0 -c:v copy"
  else
    MAP_OPT="-map 0:a"
  fi

  # 1パス目
  json=$($FFMPEG -i "$f" \
    -map 0:a \
    -af loudnorm=I=$TARGET_I:TP=$TARGET_TP:LRA=$TARGET_LRA:print_format=json \
    -f null - 2>&1)

  json_clean=$(echo "$json" | sed -n '/^{/,/}$/p')

  I=$(echo "$json_clean" | $JQ -r '.input_i')
  TP=$(echo "$json_clean" | $JQ -r '.input_tp')
  LRA=$(echo "$json_clean" | $JQ -r '.input_lra')
  THRESH=$(echo "$json_clean" | $JQ -r '.input_thresh')
  OFFSET=$(echo "$json_clean" | $JQ -r '.target_offset')

  # 2パス目
  $FFMPEG -y -i "$f" \
    $MAP_OPT \
    -map_metadata 0 \
    -af loudnorm=I=$TARGET_I:TP=$TARGET_TP:LRA=$TARGET_LRA:\
measured_I=$I:measured_TP=$TP:measured_LRA=$LRA:\
measured_thresh=$THRESH:offset=$OFFSET:linear=true \
    -ar "$OUTPUT_SR" \
    $CODEC_BASE \
    "$OUT_FILE"

  mv "$f" "$PROCESSED_DIR/"

done

echo "==== $(date) END ====" >> "$LOG_FILE"
```

## 6. 動作確認
1. ~/Audio/in に音声ファイルを投入
2. 自動処理される
3. 結果確認：

| 内容 | パス |
| ---- | ---- |
| 出力 | ~/Audio/out |
| 元ファイル | ~/Audio/processed |
| ログ | ~/Audio/log/process.log |

## 7. 設定
スクリプト冒頭に設定パラメーターをまとめてある。

   ### 7.1 OUTPUT_FORMAT
   デフォルトはFLAC。ほかwav、aac、mp3に対応

   ### 7.2 TARGET_I
   LUFSを設定。デフォルトは-16。任意の数値を指定

   ### 7.3 SAMPLE_RATE_MODE
   デフォルトはauto入力ファイルのサンプルレートを保持)。fixedでFORCE_SAMPLE_RATEで指定したサンプルレートに変換

   ### 7.4 FORCE_SAMPLE_RATE
   SAMPLE_RATE_MODEがfixedの時のサンプルレート。44100、48000、88200、9600、176400、192000に対応している(はず)。

   ※ffmpegのラウドネスノーマライズ機能がいったん192kHzにアップサンプリングしてから処理する仕様のため。

   ### 7.5 BIT_DEPTH
   出力ファイルのビットデプス。デフォルトは24(レベル操作に伴う量子化ノイズ発生を回避)

   ### 7.6 KEEP_ARTWORK
   ジャケ写データを削除する/しない。デフォルトはfalse(削除)

