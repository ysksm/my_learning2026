# Step 03: 座標変換 — translate / scale / rotate と save / restore

Step04 の「仮想座標(パン・ズーム)」の土台になる、最重要ステップです。
ここが腹落ちすると Canvas の世界が一気に楽になります。

## 1. 変換とは「これから描くものの座標の解釈を変える」こと

Canvas の変換は、**すでに描いたものには一切影響しません**。
「ペンに座標変換フィルタを装着する」イメージです。

```js
ctx.translate(100, 50); // 以降、座標に (+100, +50) が足されて解釈される
ctx.fillRect(0, 0, 10, 10); // → 実際には (100, 50) に描かれる
```

### 3つの基本変換

```js
ctx.translate(dx, dy); // 原点を (dx, dy) にずらす(平行移動)
ctx.scale(sx, sy);     // 以降の座標・サイズを sx, sy 倍にする(拡大縮小)
ctx.rotate(rad);       // 原点を中心に rad ラジアン回転(時計回り)
```

重要な性質:

- **変換は累積する**。`translate(10,0)` を2回呼ぶと合計 (20,0) ずれる
- **順番が意味を持つ**。「translate してから scale」と「scale してから translate」は結果が違う
  - `translate(100,0)` → `scale(2,2)`: 原点を 100 ずらしてから 2 倍
  - `scale(2,2)` → `translate(100,0)`: translate の 100 も 2 倍されて 200 ずれる
- `lineWidth` なども scale の影響を受ける(2倍ズームすると線も2倍太くなる)

### rotate は「原点中心」— 図形の中心で回すイディオム

`rotate` は常に **現在の原点** を中心に回ります。
「図形をその場で回したい」ときは、定番の3手順を使います:

```js
ctx.translate(cx, cy);   // 1. 原点を図形の中心に持っていく
ctx.rotate(angle);       // 2. そこで回す
ctx.fillRect(-w/2, -h/2, w, h); // 3. 中心が原点に来るように描く
```

## 2. save / restore — 状態のスタック

変換は累積するので、「ある図形のためにした変換」を元に戻さないと
次の図形がずれた場所に描かれてしまいます。そこで:

```js
ctx.save();    // 今の状態(変換 + fillStyle など全設定)をスタックに積む
// ... 変換していろいろ描く ...
ctx.restore(); // スタックから取り出して丸ごと復元する
```

- 保存されるのは **変換行列、fillStyle/strokeStyle、lineWidth、font、globalAlpha、クリップ領域など全部**。パス(下書き)は含まれない
- スタックなので **入れ子にできる**。「親の変換の中で子の変換」というツリー構造が作れる
- **鉄則: 変換をいじる関数は save で始めて restore で終わる**。
  これで関数の外に影響が漏れない(ダッシュボードの各ウィジェット描画関数はこの形にする)

```js
function drawWidget(w) {
  ctx.save();
  ctx.translate(w.x, w.y);
  // ここでは (0,0) がウィジェットの左上として描ける!
  ctx.fillRect(0, 0, w.width, w.height);
  ctx.restore();
}
```

この「translate してローカル座標で描く」パターンにすると、
ウィジェットの描画コードが自分の位置を知らなくてよくなり、部品化できます。

## 3. setTransform / resetTransform — 変換の絶対指定

`translate` 等が「今の変換に **追加**」なのに対し、`setTransform` は「**上書き**」です。

```js
ctx.setTransform(a, b, c, d, e, f); // 変換行列を直接セット
ctx.setTransform(1, 0, 0, 1, 0, 0); // = 無変換(単位行列)に戻す
ctx.resetTransform();               // 同上のショートハンド
ctx.getTransform();                 // 現在の変換を DOMMatrix で取得
```

6つの引数は 2D アフィン変換行列の成分です:

```
| a c e |   a: 横方向の倍率   c: 横方向の傾き   e: 横方向の移動
| b d f |   b: 縦方向の傾き   d: 縦方向の倍率   f: 縦方向の移動
| 0 0 1 |
```

- `translate(dx,dy)` = `setTransform(1,0,0,1,dx,dy)` を今の行列に掛けたもの
- `scale(s,s)` は a と d、`translate` は e と f に効く、と押さえておけば十分
- 深い数学は不要。「**変換の正体は 6 個の数字で、掛け算で合成される**」ことが分かれば OK

### DPR 対応との合わせ技(実戦イディオム)

Step01 の `ctx.scale(dpr, dpr)` も変換の一種です。つまり
`resetTransform()` すると DPR 対応まで消えます。実務ではこう書きます:

```js
ctx.setTransform(dpr, 0, 0, dpr, 0, 0); // 「DPR だけ効いた状態」に一発で戻す
```

毎フレームの描画の先頭でこれを呼べば、累積ミスがあってもリセットされ安全です。

## 4. 点の座標を変換で計算する — DOMMatrix

「変換後、この点はどこに行く?」を JS 側で計算したいときは DOMMatrix を使います。
(Step04 のマウス座標変換で主役になります)

```js
const m = ctx.getTransform();            // 現在の変換行列
const p = m.transformPoint(new DOMPoint(10, 20)); // (10,20) が写る先
const inv = m.inverse();                 // 逆行列 = 逆方向の変換
```

## 5. demo.html の見どころ

1. **translate の累積**: 同じ `fillRect(0,0,...)` が translate で階段状に並ぶ
2. **順番の違い**: translate→scale と scale→translate の結果比較
3. **save/restore の入れ子**: 時計の針のような親子関係
4. **回転するカード**: 「中心へ translate → rotate → 中心引き戻しで描く」イディオム

## 演習 (exercise.html)

1. `drawCard(x, y, w, h, angle)` を「save → translate → rotate → 描画 → restore」で実装する
2. その関数を使い、カードを角度違いで3枚描く(restore を忘れるとどうなるかも実験!)
3. 【発展】ループで12枚のカードを円形に並べる(時計の文字盤のように)

**完成条件**: 3枚のカードがそれぞれ指定角度で表示され、互いに影響しないこと。

## チェックポイント

- [ ] 変換が「すでに描いたもの」に影響しない理由を説明できる
- [ ] translate → scale と scale → translate の違いを説明できる
- [ ] save/restore が何を保存するか、なぜウィジェット描画関数に必須か説明できる
- [ ] `setTransform(dpr,0,0,dpr,0,0)` が何をしているか説明できる

---

## 解答例

<details>
<summary>exercise.html の解答</summary>

```js
function drawCard(x, y, w, h, angle) {
  ctx.save();
  ctx.translate(x + w / 2, y + h / 2); // 原点をカード中心へ
  ctx.rotate(angle);                   // 中心で回転
  // 中心が原点なので、左上は (-w/2, -h/2)
  ctx.beginPath();
  ctx.roundRect(-w / 2, -h / 2, w, h, 10);
  ctx.fillStyle = '#4a90d9';
  ctx.fill();
  ctx.fillStyle = '#fff';
  ctx.font = 'bold 14px sans-serif';
  ctx.textAlign = 'center';
  ctx.textBaseline = 'middle';
  ctx.fillText('CARD', 0, 0);
  ctx.restore();
}

// (2) 3枚描く
drawCard(60, 60, 140, 90, 0);
drawCard(260, 60, 140, 90, Math.PI / 12);  // 15度
drawCard(460, 60, 140, 90, -Math.PI / 8);  // -22.5度

// (3) 発展: 円形に12枚
const cx = 400, cy = 420, r = 150;
for (let i = 0; i < 12; i++) {
  const a = (i / 12) * Math.PI * 2;
  const x = cx + Math.cos(a) * r;
  const y = cy + Math.sin(a) * r;
  drawCard(x - 40, y - 25, 80, 50, a + Math.PI / 2); // 中心を向くよう回転
}
```

</details>
