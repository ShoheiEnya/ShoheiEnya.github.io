# ⑤「ゲームを改造してみよう」— アイデア例

対応ファイル：`checkpoints/05_kaizou_example.html`

## この回について

④でゲームは完成しています。この回に「正解」はありません。今日学んだ
「AIと会話しながらプログラムを直していく」という方法を使って、
みんな自由にゲームを改造してみる時間です。

`05_kaizou_example.html` は、その改造の一例として
**「ボーナスタイム」**を追加したバージョンを用意しました。手元で動かして、
「こういう改造もできるんだ」という参考にしてください。

## 追加した改造：ボーナスタイム（王様を連打！）

ラスボスを倒した後、王冠をかぶった的が中央に出てきて、
**5秒間だけ無限にクリックしてスコアを稼げる**ステージです。

### 1. 「消えない敵」を作る

体力を`Infinity`（無限）にするだけで、何回クリックしても倒れない敵になります。

```javascript
case 'king':
  this.health = Infinity;
  this.speed  = 0; // その場から動かない
  this.color  = '#ffcc33';
  this.radius = 1.5;
  break;
```

`speed: 0`にすると、通常の敵と同じ移動ロジックのまま「画面端に着く前に画面外判定に
引っかからない」ので、追加のコードなしにその場に留まる敵になります。

### 2. 王冠の見た目を組み立てる

球（土台）＋円柱（冠のバンド）＋円錐3本（トゲ）＋小さな球（宝石）を組み合わせます。
新しいモデルを作る時は、いきなり複雑な形を目指さず、簡単な図形の組み合わせから
考えると作りやすいです。

```javascript
const band = document.createElement('a-cylinder');
band.setAttribute('radius', this.radius * 0.55);
band.setAttribute('height', 0.18);
band.setAttribute('material', `color: ${gold}; shader: flat`);
this.el.appendChild(band);
```

### 3. 「ボーナスタイムだけ」5秒タイマーを追加する

ゲーム全体には時間制限がありません。ボーナスタイムが始まった時だけ、
専用の5秒タイマーを起動します。

```javascript
startBonusCountdown() {
  this.spawnKing();
  this.timeLeft = 5;
  this.timerBoxEl.style.display = ''; // このタイミングだけ「のこり」表示を見せる

  this._countdown = setInterval(() => {
    this.timeLeft--;
    this.updateHUD();
    if (this.timeLeft <= 3) this.timerBoxEl.className = 'hud-box danger'; // 残り3秒で赤く
    if (this.timeLeft <= 0) {
      clearInterval(this._countdown);
      this.winGame(); // 時間切れ＝クリア画面へ
    }
  }, 1000);
},
```

タイマー表示用のHUDボックスは④まで作っておいた`#timer-box`をそのまま再利用し、
`style.display`を使う時だけ見せる／隠す、という形にしています。
「使わない時は隠しておいて、必要な場面だけ出す」という考え方はUI作りでよく使います。

## 他にも考えられる改造アイデア

- ボーナスタイムを5秒じゃなく10秒にする、または王様以外の的も混ぜる
- ラスボスの体力や動きを変えて、もっと強く（弱く）する
- 中ボスの分裂数を2体じゃなく4体、6体にする
- ミス（外れ）した回数を数えて、多いと減点するようにする
- 効果音やBGMを追加する
- スマホでもプレイしやすいようにボタンのサイズを調整する

「こういうゲームにしたい」というイメージをAIに伝えて、実際に動かして確認し、
思った通りになっていなければまた直す——この繰り返しが、今日学んだ
「AIと一緒にプログラムを作る」ということの本質です。
