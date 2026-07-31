# ②「シューティングゲームを作ってみる」

対応ファイル：`checkpoints/02_shooting.html`

## この回で作るもの

敵が画面の左右から現れて、照準を合わせてクリックすると倒せる、
基本のシューティングゲームです。**スコアはまだ表示しません**（③で追加します）。
敵を倒しても倒し終わることはなく、ずっと敵が出続けるエンドレス仕様です。

まずはこれをAIと一緒に作って実際に動かし、お客さんに触ってもらって
「ちゃんと当たる？」「敵の動きは楽しい？」を確認するのがこの回の目的です。

## 実装ステップ

### 1. HUD・照準リング・スタート画面（HTML/CSS）

[01_start.md](01_start.md) の画面に、以下のUI要素を追加します。

```html
<!-- 照準リング（画面中央固定） -->
<div id="ring"></div>
<div id="ring-dot"></div>

<!-- 撃破時のメッセージ・ヒント -->
<div id="announce"></div>
<div id="hint">まるの中に入ったら クリック！</div>

<!-- スタート画面 -->
<div id="start-screen" class="overlay">
  <h1>🎯 まとあてシューティング</h1>
  <p>てきが まるの中に 入ったら…</p>
  <p><strong style="color:#ffff00;font-size:24px">クリック！</strong></p>
  <button id="start-btn">▶ はじめる</button>
</div>
```

リングのCSSはこの形です。敵が中に入ると `.ready` クラスが付いて緑に光ります。

```css
#ring {
  position: fixed; top: 50%; left: 50%;
  transform: translate(-50%, -50%);
  width: 200px; height: 200px;
  border: 7px solid rgba(255,255,255,0.75);
  border-radius: 50%;
  pointer-events: none; z-index: 100; display: none;
}
#ring.ready {
  border-color: #00ff44;
  box-shadow: 0 0 30px #00ff44, 0 0 60px rgba(0,255,68,0.4);
}
```

### 2. `gallery-manager` コンポーネント（ゲーム全体の管理役）

`<a-scene gallery-manager>` と書いて、シーンにゲーム管理コンポーネントを付けます。

```javascript
AFRAME.registerComponent('gallery-manager', {
  init() {
    this.score        = 0;
    this.scoreEnabled  = false; // この回はスコアなし。③でtrueにする
    this.state         = 'idle';
    this.enemies       = new Set(); // 画面上の敵を管理するSet

    this.hudEl     = document.getElementById('hud');
    this.ringEl    = document.getElementById('ring');
    this.hintEl    = document.getElementById('hint');
    this.startScreen = document.getElementById('start-screen');

    document.getElementById('start-btn')?.addEventListener('click', () => this.startGame());

    this._onClick = this._onClick.bind(this);
    document.addEventListener('mousedown', this._onClick);

    // 敵が倒された時のイベント
    this.el.addEventListener('enemy-defeated', e => {
      if (this.state !== 'playing') return;
      this.showAnnounce('やったー！🎉', '#ffff00');
    });
  },

  startGame() {
    this.state = 'playing';
    this.clearEnemies();
    this.startScreen.style.display = 'none';
    this.hudEl.style.display       = 'flex';
    this.ringEl.style.display      = 'block';
    this.hintEl.style.display      = 'block';

    this.startSpawning();
  },
  ...
```

### 3. クリック判定（画面ピクセル距離）

[00_overview.md](00_overview.md) で説明した `pxDistToCenter` を使います。

```javascript
const RING_RADIUS_PX = 100;

function pxDistToCenter(el, camera) {
  if (el._dead || !el.object3D) return Infinity;
  const wp = new THREE.Vector3();
  el.object3D.getWorldPosition(wp);
  const p  = wp.clone().project(camera);
  const dx = p.x * (window.innerWidth  / 2);
  const dy = p.y * (window.innerHeight / 2);
  return Math.sqrt(dx * dx + dy * dy);
}
```

```javascript
_onClick(e) {
  if (e.button !== 0) return;
  if (this.state !== 'playing') return;

  const camera = document.getElementById('cam')?.getObject3D('camera');
  if (!camera) return;

  let closest = null, closestDist = Infinity;
  this.enemies.forEach(el => {
    const d = pxDistToCenter(el, camera);
    if (d < RING_RADIUS_PX && d < closestDist) { closest = el; closestDist = d; }
  });

  if (closest?.components?.['gallery-enemy']) {
    closest.components['gallery-enemy'].hit();
  }
},
```

リングを光らせる方も同じ関数を使い、`tick` ではなく `tock` に書きます
（理由は[00_overview.md](00_overview.md)参照）。

```javascript
tock() {
  if (this.state !== 'playing') return;
  const camera = document.getElementById('cam')?.getObject3D('camera');
  if (!camera) return;

  let inRange = false;
  this.enemies.forEach(el => {
    if (pxDistToCenter(el, camera) < RING_RADIUS_PX) inRange = true;
  });
  this.ringEl.className = inRange ? 'ready' : '';
},
```

### 4. `gallery-enemy` コンポーネント（敵1体の動きと見た目）

```javascript
AFRAME.registerComponent('gallery-enemy', {
  schema: {
    fromLeft: { type: 'boolean', default: true },
    baseY:    { type: 'number',  default: 1.6  },
    type:     { type: 'string',  default: 'normal' }
  },

  init() {
    this.el._dead = false;
    this.dir = this.data.fromLeft ? 1 : -1;
    this.t   = 0;

    this.health = 1;
    this.speed  = 1.6 + Math.random() * 1.4;
    this.color  = '#f44336'; // ②では単色（赤）。カラフル化は③の「カラフルに」の要望で行う
    this.radius = 0.72;

    this.buildVisual();
  },

  buildVisual() {
    this.el.setAttribute('geometry', { primitive: 'sphere', radius: this.radius });
    this.el.setAttribute('material', { color: this.color, roughness: 0.35 });
    // 目を2つ追加（白目＋黒目のスフィアを4個貼るだけ）
  },

  tick(time, delta) {
    if (this.el._dead) return;
    const dt = delta / 1000;
    this.t  += dt;
    const pos = this.el.object3D.position;

    pos.x += this.dir * this.speed * dt;
    pos.y  = this.data.baseY + Math.sin(this.t * 1.8) * 0.18; // ふわふわ上下に揺れる
    if (pos.x > 14 || pos.x < -14) {
      this.el.emit('enemy-destroyed', {});
      if (this.el.parentNode) this.el.parentNode.removeChild(this.el);
    }
  },

  hit() {
    if (this.el._dead) return;
    this.health--;
    // 白フラッシュ＋一瞬つぶれるアニメーション
    if (this.health <= 0) this.die();
  },

  die() {
    this.el._dead = true;
    this.el.sceneEl.emit('enemy-defeated', {});
    this.el.emit('enemy-destroyed', {});
    setTimeout(() => { if (this.el.parentNode) this.el.parentNode.removeChild(this.el); }, 320);
  },
});
```

### 5. 敵をスポーンさせ続ける（エンドレス）

```javascript
spawnNormal() {
  const lanes = [
    { y: 0.8,  z: -8 },
    { y: 1.6,  z: -9 },
    { y: 2.35, z: -8 },
  ];
  const lane     = lanes[Math.floor(Math.random() * lanes.length)];
  const fromLeft = Math.random() > 0.5;
  this._addEnemy('normal', fromLeft ? -12 : 12, lane.y, lane.z,
    `type: normal; fromLeft: ${fromLeft}; baseY: ${lane.y}`);
},

startSpawning() {
  this.spawnNormal();
  this._spawnTimer = setInterval(() => {
    if (this.state !== 'playing') return;
    if (this.enemies.size < 3) this.spawnNormal(); // 画面上は常に3体まで
  }, 2000);
},
```

`this.enemies` は `Set`（重複しないコレクション）にして、敵が生まれたら追加、
消えたら削除する。倒すたびに新しく生まれるので、この回はゲームが終わりません。

## ポイント

- **スコアは実装するが表示しない**ようにするのがこの回のコツ。`scoreEnabled = false`
  という1つのフラグでON/OFFできるようにしておくと、③でスコアを「追加」する時に
  ロジックを書き直さずに済みます（詳しくは[03_score_design.md](03_score_design.md)）。
- 敵は `health: 1` の1回倒しタイプのみ。ボスなど複数回ヒットが必要な敵は④で追加します。
- `this.enemies.size < 3` のように**画面上の敵の数に上限を付ける**と、
  クリックが追いつかないほど敵が増え続ける事故を防げます。

## 次のステップへ

[03_score_design.md](03_score_design.md) で、スコア表示とデザインのカスタマイズを追加します。
