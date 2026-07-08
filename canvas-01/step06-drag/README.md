# Step 06: ドラッグでウィジェットを移動する

Step05 の「どれを掴んだか分かる」に、「掴んだまま動かす」を足します。
ドラッグは mousedown / mousemove / mouseup の**3イベントの連携**で、
状態機械(いまドラッグ中か?)として設計するのがポイントです。

## 1. ドラッグの基本構造

```
mousedown: ヒットテストして掴む対象を決め、「ドラッグ中」状態に入る
mousemove: ドラッグ中なら、対象の座標をマウスに追従させて再描画
mouseup:   「ドラッグ中」状態を抜ける
```

状態は変数で持ちます:

```js
let dragging = null;                 // ドラッグ中のウィジェット(null = していない)
let dragOffset = { x: 0, y: 0 };     // 後述
```

## 2. 最重要ポイント: 掴んだ位置のオフセット

素朴に `wg.x = マウス座標` とすると、掴んだ瞬間に
**ウィジェットの左上がマウスに吸い付いてワープ** します(気持ち悪い動き)。

正しくは「ウィジェットの左上から見て、どこを掴んだか」を mousedown 時に覚えておき、
移動中はその分を引きます:

```js
// mousedown 時
dragOffset.x = e.offsetX - wg.x; // 左上から掴んだ点までの距離
dragOffset.y = e.offsetY - wg.y;

// mousemove 時
wg.x = e.offsetX - dragOffset.x; // 掴んだ点がマウスに追従する
wg.y = e.offsetY - dragOffset.y;
```

図で書くと:

```
wg.x          e.offsetX(掴んだ点)
 |←― offset ―→|
 ┌────────────┬───────┐
 │            ●       │   ● を掴んだら、動かしている間も
 │      ウィジェット      │   ● がマウスの真下に居続けるべき
 └────────────────────┘
```

### 別解: movementX で差分移動

`wg.x += e.movementX` でも動きます(Step04 のパンと同じ発想)。
ただし movement 系はブラウザ/OS により精度に癖があるため、
**「mousedown 時の絶対位置 + オフセット」方式のほうが堅牢** です。本コースはこちらで統一します。

## 3. イベントはどこに張るか — window に張る理由

mousemove と mouseup を **canvas ではなく window に張る** のが実戦のコツです。

```js
canvas.addEventListener('mousedown', onDown); // 開始だけ canvas
window.addEventListener('mousemove', onMove); // 追跡は window
window.addEventListener('mouseup', onUp);
```

理由: 速くドラッグするとマウスが canvas の外に出ます。canvas に張っていると
- 外に出た瞬間 mousemove が来なくなり、ウィジェットが置き去りになる
- 外で指を離すと mouseup を取り逃し、**戻ってきたら勝手にドラッグが続く** バグになる

window に張れば外に出ても追跡が続きます。
なお window のイベントでは `offsetX` が canvas 基準にならないため、
Step05 で学んだ `clientX - getBoundingClientRect().left` 方式で座標を取ります。

```js
function getMousePos(e) {
  const rect = canvas.getBoundingClientRect();
  return { x: e.clientX - rect.left, y: e.clientY - rect.top };
}
```

## 4. requestAnimationFrame と再描画の設計

mousemove はマウスを速く動かすと **1フレームに何度も** 発火します。
そのたびに draw() すると無駄描きになるので、実戦では
**「状態だけ更新して、描画はフレームに1回」** に分離します。

```js
let needsRedraw = false;

function requestRedraw() {
  if (needsRedraw) return;   // すでに予約済みなら何もしない
  needsRedraw = true;
  requestAnimationFrame(() => {
    needsRedraw = false;
    draw();                  // 実際の描画はフレームに1回だけ
  });
}
```

- `requestAnimationFrame(fn)`: 「次の画面更新のタイミング(通常 60fps)で fn を呼んで」
  とブラウザに予約する API。描画はこれに同期させるのが基本
- イベントハンドラでは `wg.x = ...` と `requestRedraw()` だけ行う
- この分離は Step09 のダッシュボードでそのまま使います

## 5. 細かいが大事な仕上げ

```js
canvas.addEventListener('mousedown', (e) => {
  e.preventDefault();  // テキスト選択やネイティブのドラッグ開始を抑止
  ...
});
```

- ドラッグ開始時に対象を最前面へ(Step05 の splice/push)
- カーソル: 掴めるものの上では `grab`、ドラッグ中は `grabbing`
- CSS で `canvas { user-select: none; }` としておくとさらに安全

## 6. demo.html の見どころ

- 3枚のカードをドラッグで移動できる。掴んだ位置がずれない(オフセットの効果)
- 勢いよく canvas 外までドラッグしても追従が切れない(window リスナの効果)
- 画面下に「mousemove の発火回数」と「draw の実行回数」を表示
  → 速く動かすと mousemove 回数 > draw 回数 になり、間引きが効いているのが見える

## 演習 (exercise.html)

1. `getMousePos` を実装する(clientX − getBoundingClientRect)
2. mousedown: ヒットテストして `dragging` と `dragOffset` をセットする
3. mousemove: ドラッグ中の座標更新 + requestRedraw
4. mouseup: ドラッグ終了
5. 【発展】ウィジェットがキャンバスの外に出ないよう座標をクランプする
   (`Math.max(0, Math.min(W - wg.w, x))`)

**完成条件**: どこを掴んでもワープせず、外までドラッグしても壊れないこと。

## チェックポイント

- [ ] dragOffset がないとどんな見た目のバグになるか説明できる
- [ ] mousemove / mouseup を window に張る理由を2つ言える
- [ ] requestAnimationFrame は何を予約する API か説明できる
- [ ] 「状態更新」と「描画」を分離する利点を説明できる

---

## 解答例

<details>
<summary>exercise.html の解答(要点)</summary>

```js
function getMousePos(e) {
  const rect = canvas.getBoundingClientRect();
  return { x: e.clientX - rect.left, y: e.clientY - rect.top };
}

canvas.addEventListener('mousedown', (e) => {
  e.preventDefault();
  const p = getMousePos(e);
  const wg = findWidgetAt(p.x, p.y);
  if (!wg) return;
  dragging = wg;
  dragOffset = { x: p.x - wg.x, y: p.y - wg.y };
  // 最前面へ
  widgets.splice(widgets.indexOf(wg), 1);
  widgets.push(wg);
  requestRedraw();
});

window.addEventListener('mousemove', (e) => {
  if (!dragging) return;
  const p = getMousePos(e);
  // 発展: クランプ付き
  dragging.x = Math.max(0, Math.min(W - dragging.w, p.x - dragOffset.x));
  dragging.y = Math.max(0, Math.min(H - dragging.h, p.y - dragOffset.y));
  requestRedraw();
});

window.addEventListener('mouseup', () => {
  dragging = null;
});
```

</details>
