# 動画出力の仕様（背景画像を中心に）

このドキュメントは `src/index.html` の実装をそのまま書き出したものです。
「オンスクリーンのプレビュー」と「動画出力（canvas合成→MediaRecorder）」の
2つの描画経路を、背景画像の処理を中心に突き合わせます。

対象コミット時点のコードに基づく一次情報であり、推測は含みません。

---

## 1. 全体構造

このアプリには背景を描画する経路が **2つ** 存在します。

| | オンスクリーン プレビュー | 動画出力 |
|---|---|---|
| 描画方式 | CSS `background-*` (ブラウザのレイアウトエンジン) | `<canvas>` への `ctx.drawImage()` (JS自前計算) |
| 対象要素 | `#previewBg`（`.preview-stage` の子、`position:absolute;inset:0`） | `recordCanvas`（画面に表示されないオフスクリーンcanvas） |
| 制御関数 | `applyBgTransform()` | `drawCanvasBackground(ctx, w, h, scale)` |
| いつ実行される | 設定変更のたびに随時 | 録画中、毎フレーム `drawExportFrame()` から呼ばれる |

**重要**: 動画出力は「プレビューのDOM/canvasをキャプチャしている」わけではありません。
文字・EQ・背景のすべてを **JSが独自に再計算して** `recordCanvas` に一から描き直しています。
そのため、CSSの計算結果とJSの計算結果が完全に一致する保証はロジック側の責任になります。

---

## 2. キャンバス（コンテナ）サイズの決定

### 2-1. オンスクリーン: `.preview-stage` のサイズ

```css
.preview-stage{
  position:relative;overflow:hidden;width:100%;
  height:min(46vh,360px);min-height:160px;
}
.preview-stage-outer .preview-stage.ratio-vertical{
  aspect-ratio:9/16;height:min(70vh,640px);width:auto;max-width:100%;
}
.preview-stage-outer .preview-stage.ratio-horizontal{
  aspect-ratio:16/9;width:100%;max-width:720px;height:auto;
}
```

- **自由表示**: 親要素の幅いっぱい、高さは `min(46vh, 360px)`（手動リサイズ可）。
- **縦動画(9:16)**: 高さ基準 `min(70vh, 640px)`、幅は `aspect-ratio` から自動算出。
- **横動画(16:9)**: 幅基準だが **`max-width:720px` の上限あり**。
  ウィンドウが広い環境では、実際の表示幅は `min(パネル幅, 720px)` に頭打ちになります。
  高さは `aspect-ratio:16/9` から自動算出（最大 405px）。

実際の描画サイズは `getComputedStyle` 経由ではなく、
`#previewEqCanvas`（`.preview-stage` に `inset:0` で重なっているcanvas）の
`getBoundingClientRect()` から取得しています（`resizeEqCanvas()`）。

```js
function resizeEqCanvas(){
  const rect = previewEqCanvas.getBoundingClientRect();
  previewEqCanvas.width  = Math.max(1, Math.round(rect.width));
  previewEqCanvas.height = Math.max(1, Math.round(rect.height));
}
```

### 2-2. 動画出力: `recordCanvas` のサイズ

```js
const EXPORT_SHORT_EDGE = { '720p': 720, '1080p': 1080, '4k': 2160 };

function getExportResolution(){
  const shortEdge = EXPORT_SHORT_EDGE[exportResolutionSelect.value] || 720;
  if(previewStage.classList.contains('ratio-vertical')){
    return { w: shortEdge, h: Math.round(shortEdge * 16 / 9) };
  }
  if(previewStage.classList.contains('ratio-horizontal')){
    return { w: Math.round(shortEdge * 16 / 9), h: shortEdge };
  }
  // 自由表示: オンスクリーンの現在のアスペクト比を踏襲
  const curW = previewEqCanvas.width || 16, curH = previewEqCanvas.height || 9;
  const aspect = curW / curH;
  let w, h;
  if(aspect >= 1){ h = shortEdge; w = Math.round(shortEdge * aspect); }
  else { w = shortEdge; h = Math.round(shortEdge / aspect); }
  w -= w % 2; h -= h % 2;
  return { w: Math.max(2, w), h: Math.max(2, h) };
}
```

呼び出し順（`startRecording()` 内）:

```js
resizeEqCanvas();                       // ① オンスクリーンの実寸を最新化
const targetRes = getExportResolution(); // ② 出力解像度を決定
recordCanvas.width  = targetRes.w;
recordCanvas.height = targetRes.h;
```

**縦動画/横動画モード**: アスペクト比は常に厳密な 9:16 / 16:9（例: 720p→1280×720）。
オンスクリーンの実測値は使わず、固定値を使用。
→ CSSの `aspect-ratio` も同じ比率なので、**比率自体は一致するはず**。

**自由表示モード**: オンスクリーンの実測アスペクト比をそのまま踏襲。

---

## 3. 背景画像の読み込み

```js
let previewBgImageEl = null; // ImageBitmap（優先）または HTMLImageElement（フォールバック）

function loadPreviewBgImage(source){
  if(!source){ previewBgImageEl = null; return; }
  if(typeof createImageBitmap === 'function' && source instanceof Blob){
    createImageBitmap(source, { imageOrientation: 'from-image' })
      .then(function(bitmap){ previewBgImageEl = bitmap; })
      .catch(function(){ loadPreviewBgImageFallback(source); });
  } else if(typeof source === 'string'){
    loadPreviewBgImageFallback(source);
  } else {
    loadPreviewBgImageFallback(URL.createObjectURL(source));
  }
}
```

- ファイル選択時、CSS側は `previewBg.style.backgroundImage = 'url(...)'` を **即座に** 設定。
- JS側（動画出力用）は同じ `File` から非同期に `createImageBitmap()` で読み込み、
  完了後に `previewBgImageEl` へ格納。**この2つは別々の読み込み処理です。**
- `drawCanvasBackground()` は `img.complete`（Image）または常にtrue（ImageBitmap）で
  「読み込み完了」を判定し、未完了なら背景をグラデーションで代替表示。

---

## 4. 背景サイズ・位置の計算（本題）

### 4-1. オンスクリーン: CSSカスタムプロパティ経由

```js
function applyBgTransform(){
  let sizeValue;
  switch(bgSizeModeSelect.value){
    case 'cover': sizeValue = 'cover'; break;
    case 'auto':  sizeValue = 'auto'; break;
    case 'zoom':  sizeValue = bgZoomInput.value + '% auto'; break;
    default:      sizeValue = 'contain';
  }
  previewPanel.style.setProperty('--preview-bg-size', sizeValue);
  previewPanel.style.setProperty('--preview-bg-position',
    bgPosXInput.value + '% ' + bgPosYInput.value + '%');
}
```

```css
.preview-bg{
  position:absolute;inset:0;
  background-size:var(--preview-bg-size, contain);
  background-position:var(--preview-bg-position, 50% 50%);
  background-repeat:no-repeat;
  background-color:#14171c;
}
```

→ **ブラウザのCSSエンジンが `background-size`/`background-position` の仕様通りに計算**。
JS側は文字列を組み立てて渡しているだけで、比率計算はしていません。

### 4-2. 動画出力: `drawCanvasBackground()` の自前計算

```js
function drawCanvasBackground(ctx, w, h, scale){
  scale = scale || 1;
  ctx.fillStyle = '#14171c';
  ctx.fillRect(0, 0, w, h);

  const img = previewBgImageEl;
  const isBitmap = typeof ImageBitmap !== 'undefined' && img instanceof ImageBitmap;
  const ready = img && (isBitmap || (img.complete && img.naturalWidth));
  if(!ready){ /* グラデーションで代替、以降スキップ */ return; }

  const iw = isBitmap ? img.width  : img.naturalWidth;
  const ih = isBitmap ? img.height : img.naturalHeight;
  const mode = bgSizeModeSelect.value;

  let drawW, drawH;
  if(mode === 'cover'){
    const bgScale = Math.max(w / iw, h / ih);
    drawW = iw * bgScale; drawH = ih * bgScale;
  } else if(mode === 'auto'){
    drawW = iw * scale; drawH = ih * scale;   // ← オンスクリーン解像度に対する比率で補正
  } else if(mode === 'zoom'){
    const zoomRaw = parseFloat(bgZoomInput.value);
    const zoom = (isNaN(zoomRaw) ? 100 : zoomRaw) / 100;
    drawW = w * zoom; drawH = drawW * (ih / iw);
  } else { // contain
    const bgScale = Math.min(w / iw, h / ih);
    drawW = iw * bgScale; drawH = ih * bgScale;
  }

  const posXRaw = parseFloat(bgPosXInput.value);
  const posYRaw = parseFloat(bgPosYInput.value);
  const posX = (isNaN(posXRaw) ? 50 : posXRaw) / 100;
  const posY = (isNaN(posYRaw) ? 50 : posYRaw) / 100;

  ctx.drawImage(img, (w - drawW) * posX, (h - drawH) * posY, drawW, drawH);

  if(previewBg.classList.contains('has-image')){
    ctx.fillStyle = 'rgba(8,10,13,.48)';
    ctx.fillRect(0, 0, w, h);
  }
}
```

`scale` の中身（`drawExportFrame()` 内）:

```js
const scale = w / (previewEqCanvas.width || w);
// w = recordCanvas.width（出力解像度）
// previewEqCanvas.width = オンスクリーンの実測ピクセル幅（resizeEqCanvas()で直近に更新）
```

### 4-3. モード別の理論上の一致性

| モード | 計算の基準 | 出力解像度に依存するか | 理論上オンスクリーンと一致するか |
|---|---|---|---|
| **contain** | `min(w/iw, h/ih)` | ×（比率のみ） | ○ コンテナのアスペクト比が同じなら一致 |
| **cover** | `max(w/iw, h/ih)` | ×（比率のみ） | ○ 同上 |
| **zoom** | `w * (zoom/100)` | ×（`w` に対する割合） | ○ 同上 |
| **auto** | `iw * scale` | ○（`scale` で明示補正） | ○ `scale` が正しければ一致 |

**contain/cover/zoom は理論上、コンテナのアスペクト比さえ同じなら出力解像度の絶対値に
依存せず一致するはずです**（横動画/縦動画モードでは、オンスクリーンもCSSの
`aspect-ratio` で厳密に 16:9 / 9:16 に固定されているため）。

---

## 5. 検証履歴（このドキュメント作成までに実施したテスト）

Playwrightで以下を検証し、いずれも一致を確認済み：

1. 合成画像（400×300, 4:3）× 横動画 × contain・cover
2. 合成画像（1600×500, 3.2:1）× 横動画 × cover（幅がコンテナより広いケース）
3. 大きめJPEG（3000×2250）× 横動画 × contain
4. EXIF回転情報つきJPEG（Chromium / WebKit 両エンジン）
5. 正方形画像（1000×1000 相当）× 横動画 × cover × 縦位置=0
   → **ここで `parseFloat(...) || 50` のゼロ落ちバグを発見・修正済み**（コミット `fed4361`）

修正後、5のケースはオンスクリーンと出力動画で一致を再確認済み。

---

## 6. まだ検証できていない/疑わしい点

- ~~横動画モードの `max-width:720px`~~ → **検証済み**。
  ビューポート幅2400pxでも `.preview-stage` は720×405に頭打ちになるが、
  この状態でも cover + 縦位置0 のオンスクリーン/出力は一致することを確認。
- ユーザー実機（Chrome, 実際の背景写真）での `bgSizeModeSelect.value` /
  `bgPosXInput.value` / `bgPosYInput.value` / `bgZoomInput.value` の**実際の値**を
  未確認（スクリーンショットからの目視推測のみ）。
- ブラウザウィンドウの実際の横幅（`previewEqCanvas.width` の実測値）が未確認。

## 7. 次の診断ステップ（提案）

上記6を潰すため、ブラウザのDevToolsコンソールで以下を実行し、値を貼っていただくのが
最短です（録画ボタンを押す **直前** に実行）:

```js
console.log({
  mode: document.getElementById('bgSizeMode').value,
  zoom: document.getElementById('bgZoom').value,
  posX: document.getElementById('bgPosX').value,
  posY: document.getElementById('bgPosY').value,
  ratio: document.getElementById('previewStage').className,
  stageRect: document.getElementById('previewStage').getBoundingClientRect(),
  canvasSize: { w: document.getElementById('previewEqCanvas').width, h: document.getElementById('previewEqCanvas').height }
});
```
