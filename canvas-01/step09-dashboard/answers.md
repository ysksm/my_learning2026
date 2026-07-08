# Step 09 演習の実装例

まずは自力で。詰まったらヒントとして部分的に読むのがおすすめです。

## 1. Delete キーで選択中ウィジェットを削除

```js
window.addEventListener('keydown', (e) => {
  if ((e.key === 'Delete' || e.key === 'Backspace') && selected) {
    widgets.splice(widgets.indexOf(selected), 1);
    selected = null;
    hovered = null;
  }
});
```

ポイント: canvas はフォーカスを持たないので `window` に張る。
Mac には Delete キーがない(Backspace 扱い)ので両方見る。

## 2. 折れ線グラフ型 'line' を追加

`WIDGET_RENDERERS` にエントリを1つ足すだけ。他のコードは一切触らない
(部品化の効果を体感するのがこの課題の狙い)。

```js
line(wg) {
  drawCardBase(wg);
  drawTitle(wg);
  const pad = 14;
  const left = pad, right = wg.w - pad;
  const bottom = wg.h - pad, top = 46;
  const stepX = (right - left) / (wg.data.length - 1);

  ctx.beginPath();
  wg.data.forEach((v, i) => {
    const px = left + i * stepX;
    const py = bottom - (bottom - top) * (v / 100);
    i === 0 ? ctx.moveTo(px, py) : ctx.lineTo(px, py);
  });
  ctx.strokeStyle = '#d9782a';
  ctx.lineWidth = 2.5;
  ctx.lineJoin = 'round';
  ctx.stroke();
},
```

widgets 配列に追加:

```js
{ type: 'line', x: 700, y: 40, w: 300, h: 200, title: 'トラフィック',
  data: [20, 45, 30, 70, 55, 85, 60, 92] },
```

## 3. グリッドスナップ(ドラッグ終了時)

mouseup で `mode` を戻す前に丸める:

```js
window.addEventListener('mouseup', () => {
  if (mode === 'drag' && selected) {
    const GRID = 20;
    selected.x = Math.round(selected.x / GRID) * GRID;
    selected.y = Math.round(selected.y / GRID) * GRID;
  }
  mode = 'idle';
  activeHandle = null;
  resizeStart = null;
});
```

発展: ドラッグ「中」に常にスナップさせたければ mousemove 側で丸める。
リサイズにも適用するなら w/h も丸める。

## 4. ダブルクリックで新ウィジェット追加

```js
canvas.addEventListener('dblclick', (e) => {
  const s = getMousePos(e);
  const p = screenToWorld(s.x, s.y); // ワールド座標に変換してから置く
  if (findWidgetAt(p.x, p.y)) return; // 既存ウィジェット上では何もしない

  widgets.push({
    type: 'counter',
    x: p.x - 110, y: p.y - 60, // ダブルクリック位置が中心になるように
    w: 220, h: 120,
    title: '新規カウンタ',
    value: 0,
  });
});
```

## 5. localStorage への永続化

```js
const STORAGE_KEY = 'mini-dashboard-v1';

function saveState() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify({ widgets, camera }));
}

function loadState() {
  const raw = localStorage.getItem(STORAGE_KEY);
  if (!raw) return;
  try {
    const s = JSON.parse(raw);
    widgets.length = 0;            // const 配列なので中身だけ入れ替える
    widgets.push(...s.widgets);
    Object.assign(camera, s.camera);
  } catch (e) {
    console.warn('保存データが壊れています', e);
  }
}

// 保存タイミング: 操作が終わるたび + ズーム後
window.addEventListener('mouseup', saveState);
canvas.addEventListener('wheel', () => saveState(), { passive: true }); // 既存の wheel とは別に登録して OK
window.addEventListener('keydown', () => saveState());

loadState(); // 起動時に復元。loop() より前に呼ぶ
```

設計の答え合わせ: ウィジェットを **「type 文字列 + プレーンなデータ」** で持ち、
描画関数は `WIDGET_RENDERERS[type]` で引く構成にしていたのは、
まさに JSON でそのまま保存・復元できるようにするためです。
関数や DOM 参照をウィジェットオブジェクトに持たせているとここで詰みます。
