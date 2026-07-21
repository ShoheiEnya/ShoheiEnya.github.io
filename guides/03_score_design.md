# ③「スコアとデザインの追加」

対応ファイル：`checkpoints/03_score_design.html`

## この回で作るもの

②のお客さんの感想を受けて、ゲームをブラッシュアップします。

1. **スコア表示を追加**する（HUDに点数を表示し、倒すたびに加算）
2. **お客さん指定のデザイン**を反映する（敵の色・大きさ・動きなどを自由に変更）

②はまだゲームとして完成していません。この回で初めて「スコアを競う」楽しさが加わります。

## 実装ステップ

### 1. スコアのON/OFFフラグを反転する

②では `this.scoreEnabled = false;` としていた場所を `true` にするだけで、
スコア機能が有効になります。

```javascript
startGame() {
  this.score        = 0;
  this.scoreEnabled  = true; // ← ②から変えたのはこの1行だけ
  ...
  this.scoreEl.parentElement.style.display = this.scoreEnabled ? '' : 'none';
  ...
```

②のコードは最初から「スコア加算・表示処理」を `if (this.scoreEnabled) {...}` で
囲んでおいたので、フラグを変えるだけでON/OFFが切り替わります。

```javascript
this.el.addEventListener('enemy-defeated', e => {
  if (this.state !== 'playing') return;
  if (this.scoreEnabled) {
    this.score += e.detail.pts;
    this.updateHUD();
    this.showPopup(`+${e.detail.pts}`, '#ffff00', false);
    this.showAnnounce('やったー！🎉', '#ffff00');
  } else {
    this.showAnnounce('やったー！🎉', '#ffff00');
  }
});
```

### 2. HUDにスコアを表示する

```html
<div id="hud">
  <div class="hud-box">
    <div class="hud-label">スコア</div>
    <div id="score-val" class="hud-value">0</div>
  </div>
</div>
```

```javascript
updateHUD() {
  if (this.scoreEl) this.scoreEl.textContent = this.score;
},
```

### 3. 敵を倒した時の得点を決める

`gallery-enemy` の `die()` で、倒した時に何点入るかをイベントに乗せて送ります。

```javascript
die() {
  this.el._dead = true;
  const pts = 50; // 通常の敵は50点
  this.el.sceneEl.emit('enemy-defeated', { pts });
  this.el.emit('enemy-destroyed', {});
  setTimeout(() => { if (this.el.parentNode) this.el.parentNode.removeChild(this.el); }, 320);
},
```

### 4. デザインをお客さんの要望に合わせて変える

ここは「正解のコード」があるわけではなく、①のヒアリングでメモした
お客さんの要望に合わせて自由に変えるところです。変えられる場所の例：

```javascript
// 敵の色（好きな色を増やす／減らす）
const pal = ['#f44336','#ff9800','#4caf50','#2196f3','#e91e63','#9c27b0','#00bcd4'];

// 敵の大きさ
this.radius = 0.72; // 大きくすると当てやすくなる

// 敵の速さ
this.speed  = 1.6 + Math.random() * 1.4; // 数字を大きくすると速くなる

// リングの大きさ（判定はRING_RADIUS_PXと連動しているので両方変える）
#ring { width: 200px; height: 200px; }
const RING_RADIUS_PX = 100;
```

例えば「もっと的を大きくしたい」「敵をハートの形にしたい」「BGMを付けたい」など、
お客さんの要望を1つずつAIに伝えて、実際に動かして確認しながら直していきます。

## ポイント

- スコアを**後から追加する**という体験が今回のポイント。最初から全部作るのではなく、
  「まず動くものを作る → お客さんの反応を見る → 良くする」という順番で作るのが
  実際のアプリ開発でもよくある進め方だと伝えてあげると良いです。
- 見た目の変更（色・大きさ・速さ）は数値をちょっと変えるだけで印象が大きく変わるので、
  子供たちが「自分で試して確認する」体験にしやすい部分です。

## 次のステップへ

[04_boss.md](04_boss.md) で、中ボスとラスボスを追加してゲームを完成させます。
