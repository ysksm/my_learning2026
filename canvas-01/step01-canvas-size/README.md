# Step 01: キャンバスのサイズを正しく理解する

ダッシュボードの土台となるキャンバスを「にじまず・歪まず・画面いっぱいに」
表示するための最初の一歩です。ここを曖昧にすると後のすべてがぼやけます(文字通り)。

## 1. canvas には「2つのサイズ」がある

これが Step01 で一番大事な事実です。

| サイズ | 決め方 | 意味 |
|---|---|---|
| **描画バッファのサイズ** | `<canvas width="800" height="600">` 属性、または JS の `canvas.width = 800` | 内部に持っているピクセルの数(ビットマップの解像度) |
| **表示サイズ** | CSS の `width: 400px;` など | 画面上で何 px の大きさで表示するか |

- 属性を指定しないと描画バッファは **300 × 150** になります(仕様のデフォルト)
- CSS を指定しないと、表示サイズ = 描画バッファのサイズになります
- 2つが食い違うと、**ブラウザがビットマップを拡大縮小して表示** します
  - 例: バッファ 300×150 を CSS で 600×300 に表示 → 2倍に引き伸ばされて **ぼやける・歪む**

> 画像ファイル(300×150 の PNG)を `<img style="width:600px">` で表示すると
> 荒くなるのと完全に同じ現象です。canvas は「描ける画像」だと考えてください。

## 2. サイズ変更にまつわるメソッド・プロパティ

### `canvas.width` / `canvas.height` (プロパティ)

```js
canvas.width = 800;   // 描画バッファの横幅を 800px にする
```

重要な仕様: **width か height に代入すると、キャンバスの中身が全消去され、
描画状態(fillStyle や変換など)もすべてリセットされます。**
同じ値を代入し直しただけでも消えます。「サイズ変更 = 作り直し」と覚えてください。

### `canvas.clientWidth` / `canvas.clientHeight`

CSS によって決まった「実際の表示サイズ」を読み取れます(読み取り専用)。
「表示サイズに描画バッファを合わせる」処理で使います。

### `ctx.clearRect(x, y, w, h)`

指定した矩形を「完全な透明」に戻します。毎フレームの描き直しの最初に呼ぶのが定番です。

```js
ctx.clearRect(0, 0, canvas.width, canvas.height); // 全消去
```

## 3. Retina / 高DPIディスプレイ対応 — `devicePixelRatio`

Mac の Retina ディスプレイなどでは、CSS 上の 1px が物理的には 2px(以上)で表示されます。
その倍率が `window.devicePixelRatio`(以下 DPR)です。

CSS 400px 幅の場所に、バッファも 400px の canvas を置くと、
物理 800px の領域に 400px のビットマップを引き伸ばすことになり、**微妙にぼやけます**。

対策は定番パターンとして丸ごと覚えて OK です:

```js
function setupCanvas(canvas) {
  const dpr = window.devicePixelRatio || 1;
  const rect = canvas.getBoundingClientRect(); // CSS 上の表示サイズを取得

  // 描画バッファは「表示サイズ × DPR」の高解像度にする
  canvas.width  = rect.width  * dpr;
  canvas.height = rect.height * dpr;

  const ctx = canvas.getContext('2d');
  // 座標系を DPR 倍しておく。以降は CSS px 感覚で描ける
  ctx.scale(dpr, dpr);
  return ctx;
}
```

ポイント:
- バッファを DPR 倍にする → 物理ピクセルと 1:1 になり、くっきり描画される
- そのままだと座標も DPR 倍で指定する必要があって面倒なので、
  `ctx.scale(dpr, dpr)` で「1 と書いたら DPR ピクセル進む」座標系にしておく
- 結果、**コード上は CSS px だけを意識すればよくなる**

`ctx.scale` の詳細は Step03 でやります。ここでは「倍率をかける命令」とだけ理解すれば十分です。

## 4. ウィンドウリサイズへの追従

ダッシュボードは画面いっぱいに広げることが多いので、リサイズ対応は必須です。

```js
window.addEventListener('resize', () => {
  setupCanvas(canvas); // バッファサイズを取り直す(中身は消えるので…)
  draw();              // すぐ描き直す
});
```

「サイズ変更で中身が消える」仕様のため、**リサイズしたら必ず描き直す** のがセットです。
このため、実用的な canvas アプリは必ず「いつでも全体を描き直せる `draw()` 関数」を
持つ構造になります。この設計は Step06 以降でさらに重要になります。

## 5. demo.html の見どころ

ブラウザで `demo.html` を開いてください。3つのキャンバスを並べています。

1. **属性もCSSもなし** → 300×150 のデフォルトサイズ
2. **バッファ 200×100 を CSS で 400×200 に表示** → 引き伸ばされてぼやけている
3. **DPR 対応済み** → 同じ内容がくっきり表示される

ウィンドウをリサイズすると、3番のキャンバスだけが追従して描き直されます。

## 演習 (exercise.html)

`exercise.html` を開くと、ぼやけた円と文字が表示されます。
TODO コメントに従って以下を実装してください。

1. `setupCanvas()` の中身を完成させ、DPR 対応でくっきり表示させる
2. ウィンドウリサイズ時にキャンバスを追従させ、円が常に中央に来るようにする

**完成条件**: 文字と円の輪郭がくっきり表示され、ウィンドウを広げても円が中央に居続けること。

## チェックポイント(次に進む前に自問)

- [ ] `canvas.width` と CSS の `width` の違いを人に説明できる
- [ ] canvas がぼやける2大原因(CSS引き伸ばし、DPR)を言える
- [ ] `canvas.width = ...` すると何が起きるか言える(全消去 + 状態リセット)
- [ ] なぜ `draw()` 関数にまとめる設計が必要なのか説明できる

---

## 解答例

<details>
<summary>exercise.html の解答</summary>

```js
function setupCanvas() {
  const dpr = window.devicePixelRatio || 1;
  const rect = canvas.getBoundingClientRect();
  canvas.width  = rect.width * dpr;
  canvas.height = rect.height * dpr;
  ctx.scale(dpr, dpr);
}

function draw() {
  const rect = canvas.getBoundingClientRect();
  ctx.clearRect(0, 0, rect.width, rect.height);

  const cx = rect.width / 2;
  const cy = rect.height / 2;

  ctx.beginPath();
  ctx.arc(cx, cy, 80, 0, Math.PI * 2);
  ctx.fillStyle = '#4a90d9';
  ctx.fill();

  ctx.fillStyle = '#222';
  ctx.font = '20px sans-serif';
  ctx.textAlign = 'center';
  ctx.fillText('くっきり表示できた?', cx, cy + 130);
}

window.addEventListener('resize', () => {
  setupCanvas();
  draw();
});

setupCanvas();
draw();
```

</details>
