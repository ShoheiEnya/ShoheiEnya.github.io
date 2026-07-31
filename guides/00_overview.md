# まとあてシューティング — 授業用ビルドガイド 全体像

このフォルダのmdファイルは、`checkpoints/` にある5つのHTMLファイルを
「AIと一緒に、一からステップごとに作る」ための手順書です。

> ⚠️ ルート直下の `shooting-kids-guide.md` は初期バージョン（タイマー90秒・ボスHP5など）を
> 元にした古いガイドです。現在の完成版とは内容がズレているので、これから作る場合は
> このフォルダのガイドを使ってください。

> ℹ️ 同じフォルダにある [REFERENCE.md](REFERENCE.md) は別の資料です。こちらの
> `00_overview.md`〜`05_kaizou_example.md` は `checkpoints/` のHTMLを一から作るための
> 手順書ですが、`REFERENCE.md` は実際の授業で `output/game.html` を育てていく際に
> 進行役・AIが参照する詳細リファレンスです。

---

## 1. 授業の流れとファイル対応

| 授業アジェンダ | 作るもの | 対応ファイル | ガイド |
|---|---|---|---|
| ①ヒアリング | お客さんの要望メモ（コードなし） | — | — |
| ②シューティングゲームを作ってみる | 敵を撃つ基本機能（スコアなし） | `checkpoints/02_shooting.html` | [01_start.md](01_start.md) → [02_shooting.md](02_shooting.md) |
| ③スコアとデザインの追加 | スコア表示＋デザインの自由カスタマイズ | `checkpoints/03_score_design.html` | [03_score_design.md](03_score_design.md) |
| ④ボスを追加 | 中ボス（分裂）＋ラスボス（ドラゴン） | `checkpoints/04_boss.html` | [04_boss.md](04_boss.md) |
| ⑤ゲームを改造してみよう | ボーナスタイム（王様）などの発展例 | `checkpoints/05_kaizou_example.html` | [05_kaizou_example.md](05_kaizou_example.md) |

授業は講師2人（プログラマー役／お客さん役）の寸劇形式で進めます。各ステップで
お客さん役が言う要望のセリフ例・進行の流れは [demo_script.md](demo_script.md) を参照してください。

出発点は `checkpoints/01_start.html`（空・地面・雲・固定カメラだけの、なにもない3D空間）です。
ここから各ガイドの手順で機能を積み重ねていくと、最終的に `shooting-kids.html`（完成版・改造例と同内容）に辿り着きます。

---

## 2. 技術スタック

| 項目 | 内容 |
|------|------|
| エンジン | A-Frame 1.5.0（`<script src="https://aframe.io/releases/1.5.0/aframe.min.js">`） |
| 構成 | HTML・CSS・JSをすべて1ファイルに書く単一ファイル構成 |
| 操作 | マウスクリック（ポインターロックなし、カメラは完全固定） |
| 当たり判定 | 画面ピクセル距離（Raycaster不使用） |

`<a-scene>` に `gallery-manager` コンポーネントを付けてゲーム全体を管理し、
敵1体につき `gallery-enemy` コンポーネントを付けたエンティティを1つ生成する、という2コンポーネント構成が全ステップを通じて共通の骨格です。

---

## 3. 全ステップに共通する重要パターン

### パターン1：当たり判定は「ピクセル距離」で行う

敵のワールド座標をカメラに投影し、画面中心からの距離（ピクセル）が
照準リングの半径より小さいかどうかで判定します。Raycasterは使いません。

```javascript
const RING_RADIUS_PX = 100; // #ring は直径200px

function pxDistToCenter(el, camera) {
  if (el._dead || !el.object3D) return Infinity;
  const wp = new THREE.Vector3();
  el.object3D.getWorldPosition(wp);
  const p  = wp.clone().project(camera);      // ワールド座標 → NDC(-1〜1)
  const dx = p.x * (window.innerWidth  / 2);  // NDC → 画面ピクセル
  const dy = p.y * (window.innerHeight / 2);
  return Math.sqrt(dx * dx + dy * dy);
}
```

この関数はクリック判定（`_onClick`）と、リングを緑に光らせる判定（`tock`）の
両方から呼ばれます。**判定基準を1つの関数にまとめておくこと**が重要で、
別々に計算すると「リングは光るのにクリックが当たらない」というズレが起きます。

### パターン2：tick ではなく tock でUIを更新する

| | `tick` | `tock` |
|--|------|------|
| 実行タイミング | レンダリング前 | レンダリング後 |
| 使いどころ | 敵の移動など物理更新 | 照準リングの色などUI更新 |

敵の位置は `tick` で更新されるため、リングの色判定を `tick` に書くと
「1フレーム前の位置」で判定してしまいズレます。`tock` に書くことで
「実際に画面に見えている位置」と判定が一致します。

### パターン3：DOMフラグ `_dead` で状態共有する

```javascript
this.el._dead = false;                 // init()で初期化
this.el._dead = true;                  // die()で設定
if (this.el._dead) return;             // tick()の先頭でガード
this.enemies.forEach(el => {
  if (el._dead || !el.object3D) return; // Setを走査するときも必ずチェック
});
```

A-Frameのエンティティ（DOM要素）に直接フラグを持たせることで、
`gallery-manager` と `gallery-enemy` という別コンポーネント間で
「もう死んでいるか」を簡単に共有できます。

### パターン4：`scoreEnabled` フラグでスコア機能をON/OFFする

②→③でスコアを「後から追加する」ために、スコア加算処理はすべて
`if (this.scoreEnabled) { ... }` で囲んであります。`false`にすれば
見た目はそのまま、スコアだけ表示されなくなります（[02_shooting.md](02_shooting.md)参照）。

---

## 4. ローカルでの起動方法

このゲームはビルド不要の単一HTMLファイルです。ブラウザで直接開くだけで動きます。

```bash
open checkpoints/02_shooting.html
```
