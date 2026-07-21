# ①→② スタート地点を作る

対応ファイル：`checkpoints/01_start.html`

## この回で作るもの

まだ何もできない、真っ白（真っ青）な3D空間です。空・地面・雲・固定カメラだけがあり、
敵もHUDもゲームロジックも一切ありません。「①ヒアリング」でお客さんの要望を聞いたあと、
AIと一緒にコードを書き始める**出発点**になるファイルです。

## 実装ステップ

### 1. HTMLの土台

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
  <title>まとあてシューティング（スタート）</title>
  <script src="https://aframe.io/releases/1.5.0/aframe.min.js"></script>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { overflow: hidden; }
  </style>
</head>
<body>
  <!-- ここにA-Frameシーンを書く -->
</body>
</html>
```

`overflow: hidden` を忘れると、スマホなどでスクロールバーが出て画面がズレるので注意。

### 2. A-Frameシーン（空・地面・雲・固定カメラ）

```html
<a-scene id="scene" vr-mode-ui="enabled: false" renderer="antialias: true" background="color: #87ceeb">
  <a-sky id="sky" color="#87ceeb"></a-sky>

  <a-plane id="ground" rotation="-90 0 0" width="100" height="100"
    material="color: #5cb85c; roughness: 0.9"></a-plane>

  <a-entity id="clouds">
    <a-sphere position="-12 7 -22"   radius="3"   material="color: #fff; shader: flat; opacity: 0.95"></a-sphere>
    <a-sphere position="-9.5 6 -22"  radius="2"   material="color: #fff; shader: flat; opacity: 0.95"></a-sphere>
    <a-sphere position="10 7.5 -22"  radius="2.5" material="color: #fff; shader: flat; opacity: 0.95"></a-sphere>
    <a-sphere position="12.5 6.5 -22" radius="2"  material="color: #fff; shader: flat; opacity: 0.95"></a-sphere>
  </a-entity>

  <a-light type="ambient"     intensity="1.5" color="#ffffff"></a-light>
  <a-light type="directional" intensity="0.8" position="1 3 2"></a-light>

  <a-camera id="cam" position="0 1.6 0"
    look-controls="enabled: false"
    wasd-controls="enabled: false"
    user-height="0">
  </a-camera>
</a-scene>
```

## ポイント

- `look-controls="enabled: false"` と `wasd-controls="enabled: false"` で、
  マウスドラッグや矢印キーでカメラが動かないようにする。**固定カメラの的当てゲームなので、
  プレイヤーがカメラを操作できると照準の仕組みが破綻する。**
- `<a-plane>` はデフォルトで縦向き（XY平面）なので、地面にするには
  `rotation="-90 0 0"` でX軸方向に90度回転させて水平にする。
- 雲は単なる白い球（`a-sphere`）をいくつか浮かせているだけ。凝ったモデルは不要。

## 次のステップへ

ここに「敵を出す」「クリックで当てる」機能を追加していくのが
[02_shooting.md](02_shooting.md) です。
