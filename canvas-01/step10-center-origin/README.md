# Step 10: 中心原点の座標系 — キャンバス中央を (0, 0) にして管理する

Canvas の座標は標準では「左上が (0, 0)、右が +x、下が +y」です。
しかしゲージ・レーダーチャート・時計・図形エディタのような
**中心対称なもの** を作るときは、「キャンバスの中央を (0, 0)」として
座標を管理したほうが圧倒的に楽になります。このステップではその作り方を学びます。

## 1. なぜ中心を (0, 0) にしたいのか

左上原点のまま中央にウィジェットを置くと、座標が全部こうなります:

```js
// 左上原点: 「中央から左に 100」を表すのに毎回 W/2 が混ざる
ctx.arc(W / 2 - 100, H / 2, 30, 0, Math.PI * 2);
```

中心原点にすると、データは **画面サイズと無関係な純粋な位置** になります:

```js
// 中心原点: データは「中心からの相対位置」だけ
const point = { x: -100, y: 0 };
```

メリットを整理すると:

| メリット | 説明 |
|---|---|
| **対称な配置が素直に書ける** | 左右対称は `x` と `-x`、上下対称は `y` と `-y` |
| **リサイズに強い** | 中心原点なら、ウィンドウサイズが変わっても内容が自動的に中央に居続ける |
| **数学の式がそのまま使える** | 円周上の点 `(r·cosθ, r·sinθ)`、回転、極座標などが式のまま書ける |
| **データが画面から独立する** | `W/2` や `H/2` がデータに混ざらないので、保存・共有しやすい |

## 2. 仕組み: translate 一発で原点を動かす

Step03 で学んだ `translate` を使い、**描画の最初に原点を中央へ動かす** だけです。

```js
function draw() {
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0); // DPR だけの状態にリセット
  ctx.clearRect(0, 0, W, H);

  ctx.save();
  ctx.translate(W / 2, H / 2); // ← これ以降、(0, 0) はキャンバス中央
  drawScene();                 // 中身は中心原点の座標で普通に描くだけ
  ctx.restore();

  drawHud(); // HUD は変換の外 = 左上原点のまま
}
```

これは Step04 のカメラと同じ発想です。「データの座標系」と「画面の座標系」を分けて、
**変換は draw() の入口で 1 回だけかける**。描画コードにも、データにも、
`W/2` を一切書かない。ここがこのステップの核心です。

## 3. 座標変換の式

変換式は Step04 のワールド座標の式で `camera = (-W/2, -H/2)` に固定したものと同じです:

```
screenX = centerX + W / 2        centerX = screenX - W / 2
screenY = centerY + H / 2        centerY = screenY - H / 2
```

関数にしておきます。マウスイベント(スクリーン座標で届く)の解釈に必須です:

```js
function centerToScreen(cx, cy) {
  return { x: cx + W / 2, y: cy + H / 2 };
}
function screenToCenter(sx, sy) {
  return { x: sx - W / 2, y: sy - H / 2 };
}
```

```js
canvas.addEventListener('mousemove', (e) => {
  const p = screenToCenter(e.offsetX, e.offsetY);
  // p.x, p.y は「中心からどれだけ離れているか」。中央なら (0, 0)
});
```

## 4. y 軸の向きを選ぶ — Y-down か Y-up か

中心原点にしても、Canvas の y 軸は **下向きが +** のままです。
数学やグラフの感覚(上が +)にしたい場合は `scale(1, -1)` で y 軸を反転できます:

```js
ctx.translate(W / 2, H / 2);
ctx.scale(1, -1); // これ以降、上が +y(数学と同じ向き)
```

ただし **重大な副作用** があります: テキストや画像も上下反転して描かれます。
文字を描く瞬間だけ、もう一度反転して打ち消すのが定石です:

```js
function fillTextUpright(text, x, y) {
  ctx.save();
  ctx.translate(x, y); // 文字のアンカー位置へ移動してから
  ctx.scale(1, -1);    // その場で y 軸を戻す(位置は変わらない)
  ctx.fillText(text, 0, 0);
  ctx.restore();
}
```

使い分けの目安:

| 方式 | 向くケース |
|---|---|
| **Y-down(反転しない)** | UI・ダッシュボード。テキストが多いならこちら。回転角も Canvas の時計回りのまま |
| **Y-up(scale(1,-1))** | 折れ線グラフ、物理シミュレーション、数学の可視化。「値が大きい = 上」が自然なもの |

迷ったら Y-down のままで OK です。中心原点の便利さの大部分は translate だけで得られます。

## 5. リサイズに強い、の意味

左上原点では、ウィンドウを広げると内容が **左上に寄って** 右下に余白ができます。
中心原点では `translate(W / 2, H / 2)` の `W, H` がリサイズのたびに更新されるので、
**データを 1 バイトも変更せずに** 内容が中央に居続けます。

```js
window.addEventListener('resize', () => {
  setupCanvas(); // W, H を取り直す(Step01)
  draw();        // translate(W/2, H/2) が新しい中央を指す。データはそのまま
});
```

demo.html でウィンドウ幅を変えて、図形が中央に追従するのを確認してください。

## 6. Step04 のカメラと組み合わせる(知識として)

パン & ズームと中心原点は共存できます。カメラの意味を
「**画面中央** に見えているワールド座標」に変えるだけです:

```
screenX = W / 2 + (worldX - camera.x) * camera.zoom
worldX  = (screenX - W / 2) / camera.zoom + camera.x
```

```js
ctx.translate(W / 2, H / 2);            // 1. 原点を中央へ
ctx.scale(camera.zoom, camera.zoom);    // 2. ズーム
ctx.translate(-camera.x, -camera.y);    // 3. カメラ位置へ
```

この持ち方には Step04 の左上基準にない利点があります:
**「中央のズーム」が `camera.zoom *= factor` だけで済む**(補正が不要。
中央のワールド座標 = camera そのものだから、ズームしても動かない)。
Figma 的な「マウス位置中心ズーム」が必要なら Step04 の 4 手順をそのまま使います。

## 7. demo.html の見どころ

- 中央に原点マーカーと x/y 軸、対称に配置されたウィジェット(座標ラベル付き)
- 円周上に 12 個の点を `(r·cosθ, r·sinθ)` で配置 — 中心原点だから式がそのまま
- マウスを動かすと HUD に **スクリーン座標と中心座標の両方** が出る
  → 中央で (0, 0) になるのを確認すること
- 「Y-up」チェックボックスで y 軸の向きを切り替えられる
  → ラベルの文字が反転しないのは `fillTextUpright` のおかげ(コードを読むこと)
- ウィンドウ幅を変えると、データを変えずに内容が中央に追従する

## 演習 (exercise.html)

左上原点で書かれた(`W/2` まみれの)コードから始めます。

1. `screenToCenter` / `centerToScreen` を実装する
2. `draw()` に `translate(W / 2, H / 2)` を入れ、描画コードから `W/2` / `H/2` を消す
3. マウス位置の中心座標を HUD に表示する
4. 円周上の 8 個の点を `Math.cos` / `Math.sin` で配置する
5. 【発展】Y-up モード(`scale(1, -1)`)を追加し、テキストだけ正しい向きに保つ

**完成条件**: ウィンドウ幅を変えても図形が中央に居続け、
マウスがキャンバス中央のとき HUD が (0, 0) を示すこと。

## チェックポイント

- [ ] 中心原点⇔スクリーン座標の変換式を(見ないで)書ける
- [ ] 中心原点にするとリサイズに強くなる理由を説明できる
- [ ] `scale(1, -1)` の副作用(テキスト反転)と、その打ち消し方を説明できる
- [ ] 中心原点とカメラ(Step04)を組み合わせた変換 3 行を書ける

---

## 解答例

<details>
<summary>exercise.html の解答(要点)</summary>

```js
function screenToCenter(sx, sy) {
  return { x: sx - W / 2, y: sy - H / 2 };
}
function centerToScreen(cx, cy) {
  return { x: cx + W / 2, y: cy + H / 2 };
}

function draw() {
  ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
  ctx.clearRect(0, 0, W, H);

  ctx.save();
  ctx.translate(W / 2, H / 2);
  if (yUp) ctx.scale(1, -1); // 発展
  drawScene(); // 中身から W/2, H/2 を消し、中心原点の座標だけで描く
  ctx.restore();

  drawHud();
}

// 円周上の 8 点
for (let i = 0; i < 8; i++) {
  const angle = (i / 8) * Math.PI * 2;
  const x = 180 * Math.cos(angle);
  const y = 180 * Math.sin(angle);
  ctx.beginPath();
  ctx.arc(x, y, 8, 0, Math.PI * 2);
  ctx.fill();
}

// マウス位置の HUD 表示
canvas.addEventListener('mousemove', (e) => {
  mouse = screenToCenter(e.offsetX, e.offsetY);
  draw();
});

// 発展: Y-up でもテキストを正しい向きに描く
function fillTextUpright(text, x, y) {
  ctx.save();
  ctx.translate(x, y);
  if (yUp) ctx.scale(1, -1);
  ctx.fillText(text, 0, 0);
  ctx.restore();
}
```

</details>
