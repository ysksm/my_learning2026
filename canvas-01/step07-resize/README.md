# Step 07: リサイズハンドルでウィジェットの大きさを変える

ドラッグ移動(Step06)の応用です。新しい API は登場しません。
その代わり **「状態設計」が一段複雑になる** のがこのステップの学びどころです。

## 1. 仕組みの全体像

選択中のウィジェットの四隅・四辺に小さな四角(**ハンドル**)を描き、
ハンドルを掴んだときだけ「移動」ではなく「リサイズ」モードに入ります。

```
mousedown:
  1. まずハンドルにヒットテスト → 当たれば「リサイズ開始」
  2. 外れたら本体にヒットテスト → 当たれば「移動開始」(Step06)
  3. どちらでもなければ選択解除
```

**ハンドルの判定を本体より先に行う** のがポイントです。
ハンドルは本体の縁に重なっているので、後に判定すると本体に食われます。

## 2. ハンドルの定義と位置計算

8方向のハンドルを名前で管理します(CSS のカーソル名と揃えると楽):

```
nw ─── n ─── ne
│             │
w    (本体)    e
│             │
sw ─── s ─── se
```

```js
const HANDLE_SIZE = 8;

function getHandles(wg) {
  const { x, y, w, h } = wg;
  return [
    { name: 'nw', x: x,         y: y         },
    { name: 'n',  x: x + w / 2, y: y         },
    { name: 'ne', x: x + w,     y: y         },
    { name: 'e',  x: x + w,     y: y + h / 2 },
    { name: 'se', x: x + w,     y: y + h     },
    { name: 's',  x: x + w / 2, y: y + h     },
    { name: 'sw', x: x,         y: y + h     },
    { name: 'w',  x: x,         y: y + h / 2 },
  ];
}
```

ハンドルのヒットテストは「中心から ±(HANDLE_SIZE/2 + 余白)」の小さな矩形判定です。
**見た目より少し大きめに判定する**(+2〜4px)と掴みやすくなります。

## 3. リサイズの計算 — 「反対側の辺は動かさない」

例えば右下(se)ハンドルを掴んだら:

- 右端と下端がマウスに追従する
- **左端(wg.x)と上端(wg.y)は動かない**

```js
// se の場合
wg.w = mouse.x - wg.x;
wg.h = mouse.y - wg.y;
```

左上(nw)側は少しだけ難しくなります。左端を動かすと **x と w の両方** が変わります:

```js
// nw の場合: 「右端 = wg.x + wg.w」を固定したまま左端を動かす
const right = start.x + start.w;   // ドラッグ開始時に固定辺を記録しておく
wg.x = mouse.x;
wg.w = right - mouse.x;
```

そこで、**ドラッグ開始時の矩形(start)を丸ごと保存しておき、
毎回 start から計算し直す** のが堅牢な実装です(累積誤差も出ません):

```js
// mousedown 時
resizeStart = { x: wg.x, y: wg.y, w: wg.w, h: wg.h, mouseX: p.x, mouseY: p.y };

// mousemove 時(dx, dy = マウスの総移動量)
const dx = p.x - resizeStart.mouseX;
const dy = p.y - resizeStart.mouseY;

if (handle.includes('e')) wg.w = resizeStart.w + dx;             // 右辺を動かす
if (handle.includes('s')) wg.h = resizeStart.h + dy;             // 下辺を動かす
if (handle.includes('w')) { wg.x = resizeStart.x + dx; wg.w = resizeStart.w - dx; }
if (handle.includes('n')) { wg.y = resizeStart.y + dy; wg.h = resizeStart.h - dy; }
```

`'ne'.includes('n')` が true になることを利用して、
8方向を「n/s/e/w の4条件の組み合わせ」に還元しているのがミソです。

## 4. 最小サイズのクランプ

制限しないと幅 0 や **負の幅** になり、描画も判定も壊れます。

```js
const MIN = 60;
if (wg.w < MIN) {
  if (handle.includes('w')) wg.x = resizeStart.x + resizeStart.w - MIN; // 左辺側は x も直す
  wg.w = MIN;
}
```

「w だけでなく、動かしている辺が west/north のときは x/y も補正する」のを忘れずに。

## 5. カーソルのフィードバック

ハンドルにホバーしたら方向に合ったリサイズカーソルにします。
CSS カーソル名は `nw-resize`, `n-resize`, ... とハンドル名そのままです:

```js
canvas.style.cursor = handle ? `${handle.name}-resize` : 'default';
```

(ハンドル名を CSS カーソルに合わせたのはこのためです)

## 6. 状態機械としての整理

Step06 から状態が増えました。整理すると:

```js
let mode = 'idle';   // 'idle' | 'drag' | 'resize'
let target = null;    // 対象ウィジェット
let activeHandle = null; // リサイズ中のハンドル名
```

mousemove の処理は `mode` で分岐するだけになり、見通しが保てます。
Step09 ではさらに 'pan' モードが加わりますが、構造は同じです。

## 7. demo.html の見どころ

- クリックで選択 → ハンドル8個が表示される
- 各ハンドルでリサイズ。左上側でも自然に動く(start からの再計算)
- 最小サイズで止まる。カーソルが方向に応じて変わる

## 演習 (exercise.html)

1. `getHandles` と `findHandleAt` を実装する
2. mousedown の優先順位(ハンドル → 本体 → 選択解除)を実装する
3. se(右下)ハンドルのリサイズを実装する
4. 残り7方向を includes 方式で実装する
5. 【発展】最小サイズ 60px のクランプ(w/n 側の座標補正も)

**完成条件**: 8方向すべてで自然にリサイズでき、最小サイズで止まること。

## チェックポイント

- [ ] ハンドルを本体より先に判定する理由を説明できる
- [ ] nw リサイズで x と w の両方が変わる理由を図で説明できる
- [ ] 「開始時の矩形から毎回計算し直す」方式の利点を2つ言える(固定辺が守られる、累積誤差がない)
- [ ] mode 変数で状態を分ける設計の利点を説明できる

---

## 解答例

<details>
<summary>exercise.html の解答(要点)</summary>

```js
function findHandleAt(wg, x, y) {
  const R = HANDLE_SIZE / 2 + 3; // 見た目より少し大きく判定
  return getHandles(wg).find(h =>
    x >= h.x - R && x <= h.x + R && y >= h.y - R && y <= h.y + R
  ) || null;
}

canvas.addEventListener('mousedown', (e) => {
  e.preventDefault();
  const p = getMousePos(e);

  // 1. ハンドル優先
  if (selected) {
    const h = findHandleAt(selected, p.x, p.y);
    if (h) {
      mode = 'resize';
      activeHandle = h.name;
      resizeStart = { x: selected.x, y: selected.y, w: selected.w, h: selected.h,
                      mouseX: p.x, mouseY: p.y };
      return;
    }
  }
  // 2. 本体
  const wg = findWidgetAt(p.x, p.y);
  if (wg) {
    selected = wg;
    mode = 'drag';
    dragOffset = { x: p.x - wg.x, y: p.y - wg.y };
  } else {
    selected = null; // 3. 選択解除
  }
  requestRedraw();
});

// mousemove 内の resize 処理
const dx = p.x - resizeStart.mouseX;
const dy = p.y - resizeStart.mouseY;
const wg = selected, s = resizeStart, MIN = 60;

if (activeHandle.includes('e')) wg.w = s.w + dx;
if (activeHandle.includes('s')) wg.h = s.h + dy;
if (activeHandle.includes('w')) { wg.x = s.x + dx; wg.w = s.w - dx; }
if (activeHandle.includes('n')) { wg.y = s.y + dy; wg.h = s.h - dy; }

// 発展: クランプ
if (wg.w < MIN) {
  if (activeHandle.includes('w')) wg.x = s.x + s.w - MIN;
  wg.w = MIN;
}
if (wg.h < MIN) {
  if (activeHandle.includes('n')) wg.y = s.y + s.h - MIN;
  wg.h = MIN;
}
```

</details>
