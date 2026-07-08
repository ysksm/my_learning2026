# Step 02: 描画の基本 — 矩形・パス・テキストと「描画状態」

ダッシュボードのウィジェットは、結局のところ「矩形 + 角丸 + 線 + テキスト」の
組み合わせです。このステップでは基本図形の描き方と、Canvas の根本的な仕組みである
**「描画状態(ステート)」** を理解します。

## 1. Canvas 2D の頭の中のモデル

`ctx`(2Dコンテキスト)は、**設定値の束(状態)を持ったペンを1本持っている** と
イメージしてください。

- `ctx.fillStyle = 'red'` は「塗り色を赤に **設定** する」だけ。まだ何も描かれない
- `ctx.fillRect(...)` などの **描画命令を呼んだ瞬間**、その時点の設定で描かれる
- 設定は上書きするまで **ずっと残る**(これが後でバグの温床になる → save/restore は Step03)

「設定してから描く」「設定は残り続ける」——この2点がすべての基本です。

## 2. 矩形系メソッド(ダッシュボードの主役)

```js
ctx.fillRect(x, y, w, h);    // 塗りつぶした矩形を描く
ctx.strokeRect(x, y, w, h);  // 枠線だけの矩形を描く
ctx.clearRect(x, y, w, h);   // その範囲を透明に消す
```

- `(x, y)` は **左上角**。Canvas の座標系は「右が +x、**下が +y**」(数学と y が逆)
- この3つはパス(後述)を経由しない即時描画。手軽なので多用します

## 3. パス — 自由な形を描く仕組み

矩形以外の形(円、折れ線グラフ、角丸…)は「パス」で描きます。
パスとは **「輪郭の下書き」** で、作っただけでは画面に何も出ません。

```js
ctx.beginPath();              // 下書きを白紙にする(超重要)
ctx.moveTo(50, 50);           // ペンを持ち上げて (50,50) に移動
ctx.lineTo(150, 100);         // (150,100) まで線の下書き
ctx.arc(100, 100, 40, 0, Math.PI * 2); // 円弧の下書き(中心x, 中心y, 半径, 開始角, 終了角)
ctx.closePath();              // 始点まで線を引いて閉じる(任意)

ctx.stroke();                 // ← ここで初めて「輪郭線」が描かれる
ctx.fill();                   // ← 「塗りつぶし」で描かれる(同じパスに両方してよい)
```

### よくあるバグ: `beginPath()` を忘れる

パスは `beginPath()` を呼ぶまで **どんどん蓄積されます**。
ループで円を10個描くとき `beginPath()` を忘れると、
「過去の円のパスも毎回まとめて再描画」されて、色が混ざったり異常に濃くなったりします。
**「図形を描き始める前に必ず beginPath()」** を体に叩き込んでください。

### 角丸矩形 `roundRect`(ダッシュボード頻出)

```js
ctx.beginPath();
ctx.roundRect(x, y, w, h, 8);          // 角丸半径 8px の下書き
ctx.roundRect(x, y, w, h, [8, 8, 0, 0]); // 上だけ丸める(左上,右上,右下,左下)
ctx.fill();
```

モダンブラウザなら使えます。ウィジェットのカード見た目はほぼこれです。

## 4. スタイル設定(ペンの設定値たち)

```js
ctx.fillStyle   = '#4a90d9';           // 塗り色。CSS の色表記が全部使える
ctx.fillStyle   = 'rgba(255,0,0,0.5)'; // 半透明も(詳細は Step08)
ctx.strokeStyle = '#333';              // 線の色
ctx.lineWidth   = 2;                   // 線の太さ(px)
ctx.lineJoin    = 'round';             // 線の角の形: miter | round | bevel
ctx.lineCap     = 'round';             // 線の端の形: butt | round | square
```

### グラデーション(グラフの見栄えに)

```js
const g = ctx.createLinearGradient(0, 0, 0, 200); // 始点(0,0)→終点(0,200) の直線方向
g.addColorStop(0, '#4a90d9');   // 0 = 始点の色
g.addColorStop(1, '#1a4a7a');   // 1 = 終点の色
ctx.fillStyle = g;              // グラデーションも fillStyle に入れられる
```

## 5. テキスト

```js
ctx.font         = 'bold 16px sans-serif'; // CSS の font と同じ書式
ctx.textAlign    = 'center';  // x 座標をどこに合わせるか: left | center | right
ctx.textBaseline = 'middle';  // y 座標をどこに合わせるか: top | middle | alphabetic | bottom
ctx.fillText('CPU 使用率', x, y);        // 塗りテキスト(通常はこちら)
ctx.strokeText('縁取り', x, y);          // 輪郭だけ

const m = ctx.measureText('CPU 使用率'); // 描かずに幅を測る
console.log(m.width);                    // → ラベルの背景サイズ計算などに使う
```

`textBaseline` のデフォルトは `alphabetic`(欧文ベースライン)で、
日本語だと「思ったより上に描かれる」原因になります。
**迷ったら `textBaseline = 'middle'` + 中心の y 座標指定** が扱いやすいです。

## 6. 描画順序 = 重なり順序

Canvas に「レイヤー」はありません。**後から描いたものが上に重なる** だけです。
ダッシュボードでは「背景 → グリッド → ウィジェット → 選択枠」の順に描くことで
重なりを制御します(Step09 で活きてきます)。

## 7. demo.html の見どころ

ミニ「ウィジェットカード」を1枚、基本図形だけで描いています。

- 角丸カード(`roundRect` + 塗り + 枠線)
- タイトルテキスト(`textAlign` / `textBaseline` の効果をコメントで確認)
- 棒グラフ(`fillRect` のループ + グラデーション)
- 折れ線グラフ(`beginPath` + `moveTo` / `lineTo`)

## 演習 (exercise.html)

「CPU 使用率カード」を完成させます。

1. 角丸のカード背景を描く(塗り + 枠線)
2. タイトル「CPU」を左上に描く
3. データ配列から棒グラフを描く(`fillRect` をループ)
4. 【発展】80 以上の値の棒だけ色を赤にする

**完成条件**: demo と見比べて同じ構造のカードが表示されること。

## チェックポイント

- [ ] 「設定してから描く。設定は残る」を説明できる
- [ ] `beginPath()` を忘れると何が起きるか説明できる
- [ ] `fill()` と `stroke()` の違い、パスと即時描画(fillRect)の違いが分かる
- [ ] `textBaseline` がなぜ重要か分かる

---

## 解答例

<details>
<summary>exercise.html の解答(draw 関数)</summary>

```js
function draw() {
  ctx.clearRect(0, 0, W, H);

  const card = { x: 40, y: 40, w: 320, h: 220 };

  // (1) カード背景
  ctx.beginPath();
  ctx.roundRect(card.x, card.y, card.w, card.h, 12);
  ctx.fillStyle = '#ffffff';
  ctx.fill();
  ctx.strokeStyle = '#c0c8d0';
  ctx.lineWidth = 1.5;
  ctx.stroke();

  // (2) タイトル
  ctx.fillStyle = '#334';
  ctx.font = 'bold 16px sans-serif';
  ctx.textAlign = 'left';
  ctx.textBaseline = 'top';
  ctx.fillText('CPU', card.x + 16, card.y + 14);

  // (3)(4) 棒グラフ
  const data = [35, 60, 82, 45, 95, 70, 55];
  const chartX = card.x + 16;
  const chartBottom = card.y + card.h - 16;
  const chartH = 140;
  const barW = 32;
  const gap = 10;

  data.forEach((v, i) => {
    const barH = chartH * (v / 100);
    ctx.fillStyle = v >= 80 ? '#d9534f' : '#4a90d9'; // (4) 発展
    ctx.fillRect(chartX + i * (barW + gap), chartBottom - barH, barW, barH);
  });
}
```

</details>
