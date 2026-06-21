# CLAUDE.md — 声チェッカー (meeting-voice-check)

オンライン会議で「自分の声が相手にどう聞こえているか」をブラウザだけで録音・診断するクライアント完結の Web アプリ。
公開: https://snaka.github.io/meeting-voice-check/ （GitHub Pages, public, `main` ブランチのルートを配信）

## 構成

- `index.html` — アプリ本体（HTML + CSS + JS をすべてインライン）。ビルド不要・外部依存なし。
- `ref-voice.mp3` — 「お手本」音声（夏目漱石『吾輩は猫である』冒頭。`say -v Kyoko` で生成）。
- `.nojekyll` — 素の HTML をそのまま配信させる。
- 音声・診断データは外部送信せず、ブラウザのメモリ内のみで処理（永続保存なし。`localStorage` も不使用）。
- アクセス解析のみ **Google Analytics (GA4)** を利用（測定ID `G-7S9C579VP5`、`<head>` の gtag.js）。音声・診断結果は GA に送らない。プライバシー文言（ヘッダー）はこの事実と一致させること。

UI は **3ステップのウィザード**（① 録音 → ② 診断結果 → ③ シェア、`gotoStep(n)` / `.step[data-step]` で制御）。

## 開発・動作確認

```sh
python3 -m http.server 8753      # file:// では fetch/getUserMedia が動かないので必ず http で
# → http://localhost:8753/
```

JS の構文チェック（ビルドが無いので手動）:

```sh
node -e 'const fs=require("fs");const m=fs.readFileSync("index.html","utf8").match(/<script>([\s\S]*?)<\/script>/);fs.writeFileSync("/tmp/vc.js",m[1]);'
node --check /tmp/vc.js
```

ブラウザ動作確認は chrome-devtools 等で。`analyze()` / `scoreXxx()` / `drawSticker()` などは window グローバルなので、合成信号を流して評価値を直接検証できる。

デプロイ: `main` に push すると GitHub Pages が自動再ビルド（反映に約1分）。

## ⚠️ 重要な落とし穴

- **Canvas に CSS 変数は使えない**。`ctx.fillStyle = "var(--good)"` は無効化され、直前の色にフォールバックする（このバグで波形が見えない／ステッカーの色が出ない事故が2回発生）。Canvas 描画には必ず実 HEX を返す `scoreHex(score)` を使う。DOM 要素の色は `scoreColor(score)`（`var(--*)` を返す）でよい。**この2つを取り違えないこと。**
- **測定対象は「送信前のローカルなマイク信号」**。ネットワーク由来の「とぎれ／ドロップアウト」は手元の録音に現れないため原理的に評価不能（軸として持たない）。
- `getUserMedia` は既定で `echoCancellation/noiseSuppression/autoGainControl` を **すべて false**（生信号を測るため）。フィードバックテストは音響結合を測る目的で必ず false 固定。
- `file://` 直開きは不可（`fetch` で `ref-voice.mp3` が読めず、`getUserMedia` も制限）。配信前提。
- 「お手本を再生」は **`ref-voice.mp3` を `decodeAudioData` したサンプルそのもの**を再生する（波形と同一データ）。ブラウザ TTS(`SpeechSynthesis`)は使っていない。

## 評価ロジック（5軸）

`analyze(samples, sr)` が全軸を計算。総合点は5軸スコアの単純平均。フレーム長は 20ms。

| 軸 | 関数 | 指標 |
|---|---|---|
| 音量レベル | `scoreVolume` | 発話区間のアクティブレベル(dBFS, **エネルギー平均**) |
| ノイズ(SNR) | `scoreSNR` | 発話レベル − 無音区間ノイズフロア(下位10%) |
| 明瞭度(こもり) | `scoreClarity` ← `spectralBalance` | FFT で 2-4kHz(子音帯) / 0.3-1kHz(声の芯) のバランス(dB) |
| 音量の安定性 | `scoreStability` ← `levelStability` | 0.4秒窓ごとの発話レベルの標準偏差 |
| 音割れ | `scoreClip` | `|sample| >= 0.985` のサンプル比率＋ピーク張り付き |

### キャリブレーション定数（体感と合わない時はここを調整）

- 音量: `VOL_IDEAL_LO=-27, VOL_IDEAL_HI=-16`（満点帯）, `VOL_FLOOR=-44, VOL_CEIL=-6`, `VOL_TARGET=-20`（補正再生／お手本正規化のターゲット）。VoIP のアクティブ発話レベルに合わせてある。
- 明瞭度: `CLR_LO=-19, CLR_HI=-3, CLR_FLOOR=-31, CLR_CEIL=7`。
- SNR: 8/15/30 dB を区切りにした区分線形。助言はスコア帯から導出（ラベルと矛盾させない）。
- 安定性: ideal `<=3dB → 100`, `>=12dB → 0`。
- 音割れ: `0.5% → 0点`。

**校正の基準点**: お手本 `ref-voice.mp3` は -20dBFS 正規化後に **全5軸で100点**になる（理想の見本）。閾値を変えたら、お手本が満点を保つか・こもり/小声などの合成信号で妥当に下がるかをブラウザで確認する。

## お手本音声の再生成

声・セリフを変えるとき（macOS の `say` を使用）:

```sh
say -v Kyoko "吾輩は猫である。名前はまだ無い。どこで生れたか頓と見当がつかぬ。" -o /tmp/ref.aiff
ffmpeg -y -i /tmp/ref.aiff -ac 1 -ar 24000 -b:a 56k ref-voice.mp3
```

`loadReference()` が読み込み時に -20dBFS へ正規化するので、ファイル側の音量は気にしなくてよい。

## OGP 画像（SNS 共有用）

`ogp.png`（1200×630）は `ogp-template.html` をブラウザで 1200×630 にして撮ったもの。絵文字・日本語フォントを確実に描くため Chrome レンダリングで生成する（rsvg/ImageMagick だと絵文字が出ない）。デザインを変えたら `ogp-template.html` を編集 → http で開いて 1200×630 にリサイズ → スクショを `ogp.png` に保存。`<head>` の `og:image` 等は **絶対URL**（`https://snaka.github.io/...`）であること。

## フィードバック（エコー）テスト

`runFeedbackTest()`: テスト音(`PROBE_FREQS = [940, 1560]` Hz の2トーンAM)をスピーカーから流し、無音ベースラインと再生中でマイクの**当該周波数のパワー(ゲルツェル相当)**を比較。上昇量(dB)で判定（`<8` 低 / `8-20` 中 / `20-` 高）。AEC を切って音響結合そのものを測るため、ヘッドホン使用なら「拾わない＝正常」。

## 規約

- 言語: **UI・コード内コメント・ドキュメントは日本語**（このプロジェクトの確立した方針。ユーザーとのやり取りも日本語）。グローバル設定の「ghq 配下は英語」は、本プロジェクトの明示的な日本語方針で上書きする。
- コミットメッセージは英語。末尾に `Co-Authored-By: Claude <...>` を付ける。
- 新規外部依存を足す前にサプライチェーン安全性を確認（グローバル方針）。ただし現状は**依存ゼロ**を維持したい。
