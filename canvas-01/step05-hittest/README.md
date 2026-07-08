# Step 05: マウスイベントとヒットテスト

Canvas は DOM と違い、中に描いた図形は「ただのピクセル」です。
ボタンのように `click` を拾ってはくれません。
**「クリックされた座標にどの図形があるか」を自分で判定する** —— これがヒットテストです。

## 1. マウス座標を正しく取る

イベントの座標プロパティは複数あり、選び間違いがバグの定番です。

| プロパティ | 基準 | 用途 |
|---|---|---|
| `e.offsetX / offsetY` | **イベント対象要素の左上** | canvas 内の座標が直接取れる。基本これを使う |
| `e.clientX / clientY` | ブラウザ表示領域の左上 | 要素の位置を引いて使う(下記) |
| `e.pageX / pageY` | ページ全体の左上(スクロール含む) | 今回は使わない |

`offsetX` が使えない場面(要素の外まで追跡するドラッグなど。Step06 で遭遇します)では、
`clientX` から自分で計算します:

```js
const rect = canvas.getBoundingClientRect(); // canvas の画面上の位置とサイズ
const x = e.clientX - rect.left;
const y = e.clientY - rect.top;
```

**注意**: この座標は「CSS px」です。Step01 の DPR 対応をしていれば、
描画側も CSS px 基準なのでそのまま比較できます(DPR 対応の隠れた利点)。

## 2. 矩形のヒットテスト(ダッシュボードの基本)

ウィジェットは矩形なので、判定は単純な不等式で書けます:

```js
function hitTest(wg, x, y) {
  return x >= wg.x && x <= wg.x + wg.w &&
         y >= wg.y && y <= wg.y + wg.h;
}
```

数式ではなく **意味** で覚えてください:
「x が左端と右端の間」かつ「y が上端と下端の間」。

## 3. 重なりの解決 — 「上にあるものが勝つ」

Step02 で学んだ通り、配列の **後ろの要素ほど上に描かれます**。
クリック判定は見た目と一致させたいので、**配列の後ろから前へ** 走査して、
最初に当たったものを採用します:

```js
function findWidgetAt(x, y) {
  for (let i = widgets.length - 1; i >= 0; i--) { // 逆順 = 上にあるものから
    if (hitTest(widgets[i], x, y)) return widgets[i];
  }
  return null; // どれにも当たらなかった
}
```

「クリックしたら最前面に持ってくる」も配列操作だけで実現できます:

```js
const i = widgets.indexOf(target);
widgets.splice(i, 1);   // いったん抜いて
widgets.push(target);   // 末尾(=最前面)に足す
```

## 4. 複雑な形は `isPointInPath` に任せる

円や角丸、多角形の判定を不等式で書くのは大変です。
Canvas には「このパスの中にこの点は入っているか?」を判定する API があります:

```js
ctx.beginPath();
ctx.arc(cx, cy, r, 0, Math.PI * 2);      // パスを「作る」だけ。描かなくてよい
if (ctx.isPointInPath(x, y)) { ... }     // 塗り領域に入っているか
if (ctx.isPointInStroke(x, y)) { ... }   // 線の上か(lineWidth を考慮)
```

- **描画せずに判定だけに使ってよい**(fill/stroke 不要)のがポイント
- 引数の x, y は「現在の変換を適用した後の座標(≒物理ピクセル)」で解釈されます。
  DPR 対応や camera 変換をかけている場合は挙動が紛らわしいので、
  **本コースでは矩形は不等式、円は距離計算で判定する** 方針を推奨します:

```js
// 円のヒットテスト: 中心からの距離 <= 半径
const dx = x - cx, dy = y - cy;
const hit = dx * dx + dy * dy <= r * r; // sqrt を避けるため両辺2乗
```

## 5. パン・ズームとの合流(Step04 との接続)

camera がある世界では、ウィジェットの座標は **ワールド座標** で持っています。
一方マウスは **スクリーン座標** で届きます。なので:

> **ヒットテストの前に、マウス座標をワールド座標に変換する**

```js
canvas.addEventListener('mousedown', (e) => {
  const p = screenToWorld(e.offsetX, e.offsetY); // ← これを忘れると
  const wg = findWidgetAt(p.x, p.y);             //    ズーム中に当たり判定がずれる
});
```

これが「仮想座標系を持つアプリ」の鉄則です。demo では両方(HUD に表示)を確認できます。
今回の demo は簡単のため camera なしで作り、Step09 で合流させます。

## 6. ホバーエフェクトとカーソル

`mousemove` のたびにヒットテストし、当たっていれば見た目を変えます:

```js
canvas.addEventListener('mousemove', (e) => {
  hovered = findWidgetAt(e.offsetX, e.offsetY);
  canvas.style.cursor = hovered ? 'pointer' : 'default'; // カーソルも変える
  draw(); // hovered を反映して描き直す
});
```

「状態(hovered)を変えて draw() し直す」——
Step01 から続く「いつでも描き直せる draw()」設計がここでも効いています。

## 7. demo.html の見どころ

- 矩形・円・角丸カードが並び、ホバーで明るく、クリックで選択(枠線)される
- クリックした図形は最前面に移動する(逆順走査と splice/push)
- HUD にヒット中の図形名を表示

## 演習 (exercise.html)

1. 矩形の `hitTest` を実装する
2. `findWidgetAt` を逆順走査で実装する
3. click で選択(selected)を切り替え、選択枠を描く
4. mousemove でホバー中はカーソルを `pointer` にする
5. 【発展】クリックした図形を最前面に移動する

**完成条件**: 重なった図形の「見えている方」がクリックで選択されること。

## チェックポイント

- [ ] `offsetX` と `clientX` の違いと使い分けを説明できる
- [ ] なぜ逆順で走査するのか説明できる
- [ ] `isPointInPath` は何をする API か、描画なしで使えることを知っている
- [ ] camera があるとき、ヒットテスト前に何をすべきか言える

---

## 解答例

<details>
<summary>exercise.html の解答(要点)</summary>

```js
function hitTest(wg, x, y) {
  return x >= wg.x && x <= wg.x + wg.w &&
         y >= wg.y && y <= wg.y + wg.h;
}

function findWidgetAt(x, y) {
  for (let i = widgets.length - 1; i >= 0; i--) {
    if (hitTest(widgets[i], x, y)) return widgets[i];
  }
  return null;
}

canvas.addEventListener('click', (e) => {
  selected = findWidgetAt(e.offsetX, e.offsetY);
  if (selected) { // 発展: 最前面へ
    widgets.splice(widgets.indexOf(selected), 1);
    widgets.push(selected);
  }
  draw();
});

canvas.addEventListener('mousemove', (e) => {
  hovered = findWidgetAt(e.offsetX, e.offsetY);
  canvas.style.cursor = hovered ? 'pointer' : 'default';
  draw();
});

// draw() 内の選択枠:
if (wg === selected) {
  ctx.strokeStyle = '#2266cc';
  ctx.lineWidth = 3;
  ctx.strokeRect(wg.x - 4, wg.y - 4, wg.w + 8, wg.h + 8);
}
```

</details>
