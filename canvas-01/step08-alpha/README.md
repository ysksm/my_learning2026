# Step 08: 透過と合成 — globalAlpha / rgba / globalCompositeOperation

ダッシュボードの「今風の見た目」を支える技術です。
半透明パネル、ホバーの薄いハイライト、無効状態のグレーアウト、影——全部これです。

## 1. 透明度を指定する2つの方法

### 方法A: 色に埋め込む — `rgba()` / `hsla()` / 8桁HEX

```js
ctx.fillStyle = 'rgba(74, 144, 217, 0.5)'; // 50% 透明の青
ctx.fillStyle = 'hsla(210, 65%, 57%, 0.5)';
ctx.fillStyle = '#4a90d980';               // 8桁HEX(最後の2桁がアルファ: 80 = 約50%)
```

その色で塗るものだけが半透明になります。**個別の図形に使う**のに向きます。

### 方法B: 描画全体に掛ける — `ctx.globalAlpha`

```js
ctx.globalAlpha = 0.3;  // 以降のすべての描画が 30% の不透明度になる
ctx.fillRect(...);      // 半透明で描かれる
ctx.drawImage(...);     // 画像も文字も、なんでも半透明になる
ctx.globalAlpha = 1.0;  // 戻し忘れに注意!(save/restore で守るのが安全)
```

- 色に関係なく **描画命令すべて** に効く
- rgba と **掛け算で合成** される(globalAlpha 0.5 × rgba の 0.5 = 実効 0.25)
- 用途: 「ウィジェット丸ごと半透明」「ドラッグ中のゴースト表示」「フェードイン」

### 使い分けの指針

| やりたいこと | 使うもの |
|---|---|
| 特定の塗り・線だけ半透明 | rgba |
| 複数の描画命令からなる部品を丸ごと半透明 | globalAlpha(+ save/restore) |

### 落とし穴: 部品を丸ごと半透明にするときの「重なり」

半透明の図形同士が重なると、**重なった部分だけ濃く** なります。
「カード(背景 + 枠 + 文字)を丸ごと 50% にしたい」とき、
パーツごとに rgba を使うと重なり部分の濃度が破綻します。
globalAlpha を部品の描画関数全体に掛けるのが正解です
(完璧にやるなら一旦別キャンバスに描いて合成しますが、ダッシュボード用途では globalAlpha で十分)。

## 2. `clearRect` と「透明」の正体

Canvas のピクセルは RGBA で、`clearRect` は「**透明な黒 rgba(0,0,0,0)** で埋める」操作です。

- canvas 要素自体はデフォルトで透明 → 透明部分は **背後の CSS 背景が透ける**
- demo では CSS でチェッカー模様を canvas の背景に敷いて、透明が「見える」ようにしています
- `fillRect` で背景色を塗るのと `clearRect` の違い: 前者は不透明ピクセル、後者は透明ピクセル

## 3. 影 — `shadow*` プロパティ

厳密には透過機能ではありませんが、半透明色と組み合わせて使う定番なのでここで:

```js
ctx.shadowColor = 'rgba(0, 0, 0, 0.3)'; // 影は半透明の黒にするのがコツ
ctx.shadowBlur = 16;                     // ぼかし半径
ctx.shadowOffsetX = 0;
ctx.shadowOffsetY = 6;                   // 下方向に落とす
ctx.fillRect(...);                       // ← この描画に影が付く

// 影も「状態」なので残り続ける。save/restore か shadowColor='transparent' で解除
```

カードに影を付けるだけでダッシュボードの見た目が一段上がります。

## 4. `globalCompositeOperation` — 重ね方のルールを変える

通常、描画は「上に重ねる」(`source-over`、デフォルト)ですが、
このプロパティで **重ね方そのもの** を変えられます。26種類ありますが、
ダッシュボードで実用的なものだけ:

```js
ctx.globalCompositeOperation = 'source-over';      // デフォルト: 上に重ねる
ctx.globalCompositeOperation = 'destination-over'; // 「下に」描く(後から背景を敷く)
ctx.globalCompositeOperation = 'destination-out';  // 描いた形で「消しゴム」をかける
ctx.globalCompositeOperation = 'lighter';          // 色を加算(グロー・光の表現)
ctx.globalCompositeOperation = 'multiply';         // 乗算(暗くなる。オーバーレイ表現)
```

覚え方: `source` = これから描くもの、`destination` = すでに描かれているもの。

### 実用例: スポットライト(オーバーレイに穴を開ける)

「画面全体を半透明の黒で覆い、注目箇所だけくり抜く」チュートリアル UI の定番:

```js
ctx.fillStyle = 'rgba(0, 0, 0, 0.55)';
ctx.fillRect(0, 0, W, H);                        // 全体を暗くする
ctx.globalCompositeOperation = 'destination-out'; // 消しゴムモード
ctx.beginPath();
ctx.arc(cx, cy, 80, 0, Math.PI * 2);
ctx.fill();                                       // → 円形に「透明の穴」が開く
ctx.globalCompositeOperation = 'source-over';     // 必ず戻す!
```

これも「状態」です。**戻し忘れると以降の描画が全部消しゴムになります**。
save/restore で囲むのが安全です(globalAlpha も composite も save 対象)。

## 5. demo.html の見どころ

1. rgba と globalAlpha の掛け算、半透明同士の重なり比較
2. 「カード丸ごと半透明」: rgba 個別指定 vs globalAlpha の見た目の違い
3. 影付きカード
4. スポットライト(destination-out)と lighter のグロー
5. スライダーで globalAlpha をリアルタイムに変えられる

## 演習 (exercise.html)

半透明を使った「通知トースト」と「無効状態カード」を作ります。

1. カードの上に半透明白 `rgba(255,255,255,0.6)` を重ねて「無効状態」を表現する
2. `globalAlpha` + save/restore で、トースト(背景+文字+アイコン)を丸ごと 85% にする
3. トーストに影を付ける
4. 【発展】スライダーの値でトーストの globalAlpha を変える(フェード)

**完成条件**: 無効カードは文字まで薄く見え、トーストは重なり破綻なく半透明であること。

## チェックポイント

- [ ] rgba と globalAlpha の違いと使い分けを説明できる
- [ ] 「部品丸ごと半透明」で rgba 個別指定だと何が破綻するか説明できる
- [ ] clearRect が何色で埋めるのか正確に言える
- [ ] destination-out が何をするか、なぜ戻し忘れが危険か説明できる

---

## 解答例

<details>
<summary>exercise.html の解答(要点)</summary>

```js
// (1) 無効状態: 上から半透明の白を重ねるだけ
ctx.beginPath();
ctx.roundRect(card.x, card.y, card.w, card.h, 10);
ctx.fillStyle = 'rgba(255, 255, 255, 0.6)';
ctx.fill();

// (2)(3) トースト丸ごと半透明 + 影
function drawToast(x, y, alpha) {
  ctx.save();
  ctx.globalAlpha = alpha;          // 部品全体に一括で効く

  ctx.shadowColor = 'rgba(0,0,0,0.35)';
  ctx.shadowBlur = 14;
  ctx.shadowOffsetY = 5;
  ctx.beginPath();
  ctx.roundRect(x, y, 300, 60, 10);
  ctx.fillStyle = '#2b3a4a';
  ctx.fill();
  ctx.shadowColor = 'transparent';  // 文字に影が付かないよう解除

  ctx.beginPath();
  ctx.arc(x + 30, y + 30, 12, 0, Math.PI * 2);
  ctx.fillStyle = '#5aa05a';
  ctx.fill();

  ctx.fillStyle = '#fff';
  ctx.font = '14px sans-serif';
  ctx.textAlign = 'left';
  ctx.textBaseline = 'middle';
  ctx.fillText('保存しました', x + 55, y + 30);

  ctx.restore();                    // globalAlpha も影もここで元通り
}

// (4) スライダー連動
slider.addEventListener('input', () => {
  toastAlpha = Number(slider.value);
  draw();
});
```

</details>
