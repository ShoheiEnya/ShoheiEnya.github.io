# ④「ボスを追加」

対応ファイル：`checkpoints/04_boss.html`

## この回で作るもの

③までは「敵をずっと倒し続けるだけ」のゲームでした。この回でようやく
**ゲームに終わり（クリア）**ができます。

1. 通常の敵を5体倒す
2. 中ボスが出てきて、倒すと2体に分裂する → その2体も倒す
3. ラスボス（ドラゴン）が登場、HPバーを削り切ったら**ゲームクリア**

お客さんに喜んでもらえるように、ボスの見た目や技は自分なりに工夫してOKです。

> ⚠️ **中ボスとラスボスは別々のAI指示として実装すること。**
> `checkpoints/04_boss.html` にはこの2つ（中ボスの分裂＋ラスボスのドラゴン）が
> 1ファイルにまとまって入っていますが、実際にAIへ指示を出すときは
> 「中ボスを実装して」→（ß動作確認）→「ラスボスを実装して」の**2回の別リクエスト**に
> 分けます。中ボスだけを頼まれた時に、チェックポイントに見えているからといって
> ラスボスまで先回りして実装しないよう注意してください。

## 実装ステップ

### 1. ステージを管理すßる変数を用意する

```javascript
init() {
  this.stage     = 1; // 1:通常敵 → 2:中ボス → 3:ラスボス
  this.killCount = 0; // ステージ1の撃破カウント
  ...
```

### 2. ステージ1：通常の敵を10体倒したらステージ2へ

```javascript
this.el.addEventListener('enemy-defeated', e => {
  ...
  if (this.stage === 1) {
    this.killCount++;
    if (this.killCount >= 10) {
      clearInterval(this._spawnTimer);
      setTimeout(() => { if (this.state === 'playing') this.startStage2(); }, 800);
    }
  }
});
```

### 3. ステージ2：中ボスが分裂する

中ボスは`gallery-enemy`に`type: 'midBoss'`を渡した特別な敵です。倒された時、
ただ消えるのではなく「分裂イベント」を発火させます。

```javascript
// gallery-enemy.die() の中
if (this.data.type === 'midBoss') {
  this.el.sceneEl.emit('midboss-split', {
    x: this.el.object3D.position.x,
    y: this.el.object3D.position.y,
    z: this.el.object3D.position.z
  });
  this.el.emit('enemy-destroyed', {});
}
```

`gallery-manager` 側でこのイベントを受け取り、同じ場所からミニ敵を2体スポーンさせます。

```javascript
this.el.addEventListener('midboss-split', e => {
  this.spawnMinis(e.detail.x, e.detail.y, e.detail.z);
});

spawnMinis(x, y, z) {
  for (let i = 0; i < 2; i++) {
    const fromLeft = i === 0;
    this._addEnemy('mini', x, y, z, `type: mini; fromLeft: ${fromLeft}; baseY: ${y}`);
  }
},
```

ミニを2体とも倒したらステージ3へ進みます。**「逃げた」場合はカウントしない**のが
重要なポイントです。`enemy-destroyed`（逃げても発火）ではなく、`enemy-defeated`に
`isMini`フラグを乗せて「倒した時だけ」カウントします。

```javascript
if (this.stage === 2 && e.detail.isMini) {
  this.minisKilled++;
  if (this.minisKilled >= 2) {
    setTimeout(() => { if (this.state === 'playing') this.startStage3(); }, 1000);
  }
}
```

### 4. ステージ3：ラスボス（ドラゴン）とHPバー

ラスボスは体力8（クリックで倒すには8回ヒットが必要）にしています。数字を変えれば
簡単／難しいの調整ができます。

```javascript
case 'boss':
  this.health    = 8;
  this.speed     = 1.4;
  this.color     = '#e53935';
  this.radius    = 1.8;
  this.bounceDir = 1;
  break;
```

HPバーはヒットするたびに幅を更新します。

```html
<div id="boss-hp-wrap">
  <div id="boss-hp-label">ボス HP</div>
  <div id="boss-hp-bar"><div id="boss-hp-fill"></div></div>
</div>
```

```javascript
// _onClick の中、ボスにヒットした直後
if (comp.data.type === 'boss') {
  this.bossHpFillEl.style.width = `${Math.max(0, (comp.health / 8) * 100)}%`;
}
```

ボスの見た目は球と円柱・円錐を組み合わせたプロシージャル（コードで組み立てる）
ドラゴンです。翼は`animation`コンポーネントで羽ばたきアニメーションを付けています。

```javascript
pivot.setAttribute('animation',
  `property:rotation;from:${from};to:${to};dur:900;dir:alternate;loop:true;easing:easeInOutSine`);
```

ラスボスを倒したらゲームクリア画面を表示します。

```javascript
e.addEventListener('enemy-destroyed', () => {
  this.enemies.delete(e);
  if (this.stage === 3 && this.enemies.size === 0) {
    setTimeout(() => { if (this.state === 'playing') this.winGame(); }, 1200);
  }
});
```

### 5. ボスステージの背景を切り替える

`<a-sky>`は360度パノラマ画像専用なので、普通の画像を貼るには使えません。
代わりにカメラの正面に大きな板（`<a-plane>`）を置く方法を使います。

```html
<a-plane id="boss-bg-plane"
  position="0 1.6 -22" width="75" height="44"
  material="src: #boss-bg; shader: flat"
  visible="false"></a-plane>
```

ステージ3が始まったら、通常の空・地面・雲を隠して背景の板を表示します。

```javascript
document.getElementById('sky').setAttribute('visible', 'false');
document.getElementById('ground').setAttribute('visible', 'false');
document.getElementById('clouds').setAttribute('visible', 'false');
document.getElementById('boss-bg-plane').setAttribute('visible', 'true');
```

## ポイント

- ステージ管理は「1つの数値（`this.stage`）」で行い、`startStage1()` → `startStage2()`
  → `startStage3()` という3つの関数を順番に呼ぶだけのシンプルな作りにしています。
  複雑に見えるゲームも、実は単純な状態管理の積み重ねでできています。
- 「倒した」と「逃げた」を区別するために、専用のイベント（`enemy-defeated`の`isMini`）
  を用意するのがコツです。同じ「消えた」という結果でも、原因によって処理を分けたい時は
  イベントの種類・データを分ける、という考え方は他の場面でも役立ちます。

## 次のステップへ

ここでゲームは完成です。[05_kaizou_example.md](05_kaizou_example.md) では、
さらに遊びを追加する「改造」の例を紹介します。
