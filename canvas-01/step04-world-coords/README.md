# Step 04: 仮想座標(ワールド座標)とビューポート — パン & ズーム

ダッシュボードを「無限に広いキャンバスをカメラで覗いている」ように作るための
仕組みです。Figma や Miro の画面移動・拡大と同じ構造を最小構成で理解します。

## 1. 2つの座標系を分けて考える

| 座標系 | 何の座標か | 例 |
|---|---|---|
| **ワールド座標** | 仮想空間上の「本当の」位置。ウィジェットのデータはこちらで持つ | `widget.x = 1200` |
| **スクリーン座標** | キャンバス上の見た目の位置(CSS px)。マウスイベントはこちらで届く | `e.offsetX = 350` |

そして「カメラ」の状態を2つの値で持ちます:

```js
const camera = {
  x: 0,     // 画面左上に見えているワールド座標(パンで変わる)
  y: 0,
  zoom: 1,  // 拡大率(ズームで変わる)
};
```

## 2. 座標変換の式(このステップの核心)

**ワールド → スクリーン**(描くとき):

```
screenX = (worldX - camera.x) * camera.zoom
screenY = (worldY - camera.y) * camera.zoom
```

「カメラ位置を引いて、ズーム倍する」。それだけです。

**スクリーン → ワールド**(マウス位置を解釈するとき)は、上の式を逆に解くだけ:

```
worldX = screenX / camera.zoom + camera.x
worldY = screenY / camera.zoom + camera.y
```

関数にしておきます:

```js
function worldToScreen(wx, wy) {
  return { x: (wx - camera.x) * camera.zoom, y: (wy - camera.y) * camera.zoom };
}
function screenToWorld(sx, sy) {
  return { x: sx / camera.zoom + camera.x, y: sy / camera.zoom + camera.y };
}
```

## 3. 描画は Step03 の変換に任せる

毎点を自分で変換して描くこともできますが、Canvas の変換を使えば
**描画コードはワールド座標のまま書けます**:

```js
function draw() {
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0);  // DPR だけの状態に戻す
  ctx.clearRect(0, 0, W, H);

  ctx.save();
  ctx.scale(camera.zoom, camera.zoom);      // ズーム
  ctx.translate(-camera.x, -camera.y);      // パン
  // ↑ この2行が worldToScreen の式そのもの(順番に注意!)

  drawWorld(); // ← 中身はワールド座標で普通に描くだけ
  ctx.restore();
}
```

順番の確認(Step03 の知識):変換は「後に指定したものが先に座標へ適用」されます。
点はまず `translate` で `-camera.x` され、次に `scale` で zoom 倍される。
つまり `(wx - camera.x) * zoom` — 式と一致します。

## 4. パンの実装(ドラッグで画面を動かす)

「マウスが動いた分(スクリーン px)だけ、カメラを **逆方向** に動かす」です。
スクリーンの移動量をワールドの移動量に直すため zoom で割ります。

```js
canvas.addEventListener('mousemove', (e) => {
  if (!isPanning) return;
  camera.x -= e.movementX / camera.zoom; // 右にドラッグ → カメラは左へ
  camera.y -= e.movementY / camera.zoom;
  draw();
});
```

`e.movementX / movementY` は前回イベントからのマウス移動量です。

## 5. ズームの実装 — 「マウス位置を中心に」がキモ

単純に `camera.zoom *= 1.1` とすると、**画面左上を中心に** 拡大されてしまい、
見ていた場所がすっ飛んでいきます。Figma のように
「マウスカーソルの下にある点が、ズーム後も同じ画面位置に残る」ようにします。

考え方:
1. ズーム **前** に、マウス直下のワールド座標を求める → `before`
2. zoom を変更する
3. ズーム **後** に、同じスクリーン位置のワールド座標を求める → `after`
4. ずれた分 (`before - after`) だけカメラを動かして帳尻を合わせる

```js
canvas.addEventListener('wheel', (e) => {
  e.preventDefault(); // ページ全体のスクロールを止める

  const mouse = { x: e.offsetX, y: e.offsetY };     // スクリーン座標
  const before = screenToWorld(mouse.x, mouse.y);   // 1.

  const factor = e.deltaY < 0 ? 1.1 : 1 / 1.1;      // 上回し=拡大
  camera.zoom = Math.min(8, Math.max(0.2, camera.zoom * factor)); // 2. + 上下限

  const after = screenToWorld(mouse.x, mouse.y);    // 3.
  camera.x += before.x - after.x;                   // 4.
  camera.y += before.y - after.y;
  draw();
}, { passive: false }); // preventDefault するので passive: false が必要
```

- `e.deltaY`: ホイールの回転量。負 = 上回し
- ズームは掛け算(`*= factor`)にすると、どの倍率でも同じ「感触」になります
- 上下限(0.2〜8倍など)を必ず設ける。無限に縮小できると迷子になります

## 6. DOMMatrix を使う別解(知識として)

`ctx.getTransform().inverse().transformPoint(p)` でも screen→world 変換ができます。
回転を含む複雑な変換ではこちらが確実ですが、パン+ズームだけなら
今回の手書きの式のほうが見通しがよいので、本コースでは式で通します。

## 7. demo.html の見どころ

- グリッドとウィジェット風の矩形が並ぶ「ワールド」をドラッグでパン、ホイールでズーム
- 左上に camera の中身とマウスのワールド座標をリアルタイム表示
  → **動かしながら数値を見る** と、式の意味が体で分かります
- グリッドは「見えている範囲だけ」線を引いている(無限グリッドの定石)ので
  その計算もコメント付きで読んでみてください

## 演習 (exercise.html)

パン・ズームなしの状態から始めます。

1. `screenToWorld` / `worldToScreen` を実装する
2. `draw()` に camera の変換(scale + translate)を入れる
3. ドラッグでパンできるようにする
4. ホイールで「マウス位置中心の」ズームを実装する
5. 【発展】「Reset View」ボタンで camera を初期状態に戻す

**完成条件**: マウス直下の図形が、ズームしてもマウス直下に留まり続けること。

## チェックポイント

- [ ] ワールド座標とスクリーン座標の変換式を(見ないで)書ける
- [ ] `scale → translate` の2行がなぜ変換式と一致するのか説明できる
- [ ] パンで zoom による割り算が必要な理由を説明できる
- [ ] マウス中心ズームの4手順を説明できる

---

## 解答例

<details>
<summary>exercise.html の解答(要点)</summary>

```js
function screenToWorld(sx, sy) {
  return { x: sx / camera.zoom + camera.x, y: sy / camera.zoom + camera.y };
}
function worldToScreen(wx, wy) {
  return { x: (wx - camera.x) * camera.zoom, y: (wy - camera.y) * camera.zoom };
}

function draw() {
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
  ctx.clearRect(0, 0, W, H);
  ctx.save();
  ctx.scale(camera.zoom, camera.zoom);
  ctx.translate(-camera.x, -camera.y);
  drawWorld();
  ctx.restore();
  drawHud();
}

let isPanning = false;
canvas.addEventListener('mousedown', () => { isPanning = true; });
window.addEventListener('mouseup', () => { isPanning = false; });
canvas.addEventListener('mousemove', (e) => {
  if (!isPanning) return;
  camera.x -= e.movementX / camera.zoom;
  camera.y -= e.movementY / camera.zoom;
  draw();
});

canvas.addEventListener('wheel', (e) => {
  e.preventDefault();
  const before = screenToWorld(e.offsetX, e.offsetY);
  const factor = e.deltaY < 0 ? 1.1 : 1 / 1.1;
  camera.zoom = Math.min(8, Math.max(0.2, camera.zoom * factor));
  const after = screenToWorld(e.offsetX, e.offsetY);
  camera.x += before.x - after.x;
  camera.y += before.y - after.y;
  draw();
}, { passive: false });

// 発展
document.getElementById('reset').addEventListener('click', () => {
  camera.x = 0; camera.y = 0; camera.zoom = 1;
  draw();
});
```

</details>
