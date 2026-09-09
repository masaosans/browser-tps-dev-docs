# 🎮 ブラウザTPSゲーム — 要素分解＆最適実現方法

> **目標:** スマホブラウザで動作する3人称シューティング（TPS）ゲーム
> **技術スタック:** Three.js r162+ + nipplejs + 単一HTMLファイル
> **最終更新:** 2025-09-09（最新Web調査ベース）

---

## 📊 要素分解マトリクス

```
┌──────────────┬─────────────────────────────────────────────────┐
│   カテゴリ    │              含まれる要素                        │
├──────────────┼─────────────────────────────────────────────────┤
│ A. コアレンダリング │ Three.jsシーン, カメラ, ライティング     │
│ B. キャラクター   │ プレイヤーモデル, アニメーション, 移動      │
│ C. カメラシステム │ Spring Arm, Orbit, 衝突回避, ズーム        │
│ D. タッチ操作     │ ジョイスティック, ボタン, カメラ回転スワイプ │
│ E. 射撃システム   │ Hit Scan/Projectile, リコイル, マズルフラッシュ│
│ F. 敵AI          │ ステートマシン, 視界判定, パトロール, 攻撃    │
│ G. 衝突・物理     │ AABB, 距離計算, カバ判定                     │
│ H. ゲームシステム │ スコア, HP, ウェーブ制, リロード             │
│ I. UI/HUD        │ レスポンシブオーバーレイ, タッチボタン       │
│ J. パフォーマンス │ LOD, InstancedMesh, モバイル最適化         │
│ K. 環境・レベル   │ プロシージャル生成, カバ障害物               │
└──────────────┴─────────────────────────────────────────────────┘
```

---

## A. コアレンダリング

### 要件
- Three.js r162+ で3Dシーン描画
- モバイルブラウザ（iOS Safari / Android Chrome）で動作
- リサイズ対応 + レスポンシブ

### 最適実現方法
```javascript
// CDN読み込み（importmap方式 — 最新Three.js推奨）
<script type="importmap">
{
  "imports": {
    "three": "https://cdn.jsdelivr.net/npm/three@0.162.0/build/three.module.js",
    "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.162.0/examples/jsm/"
  }
}
</script>

// レンダラー（モバイル最適化）
const isMobile = /Android|iPhone|iPad/i.test(navigator.userAgent);
const renderer = new THREE.WebGLRenderer({ antialias: !isMobile });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // モバイルは最大2
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);

// レスポンシブ対応
window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});
```

### 調査結果
- **importmap方式**がThree.js r150+の標準。CDN経由で依存解決
- `setPixelRatio` はモバイルで最大2に制限（4xだと重い）
- アンチエイリアスはモバイルでOFF推奨（パフォーマンス影響大）

---

## B. キャラクター

### 要件
- プレイヤーキャラクターの視覚表現
- WASD/ジョイスティックでの移動
- カメラ基準の相対移動（TPS特有）
- 地面衝突判定

### 最適実現方法
```javascript
// 簡易キャラクター（CapsuleGeometry — Three.js標準）
const playerGroup = new THREE.Group();

// 胴体
const bodyGeo = new THREE.CapsuleGeometry(0.4, 1.2, 4, 8);
const bodyMat = new THREE.MeshStandardMaterial({ color: 0x3498db });
const body = new THREE.Mesh(bodyGeo, bodyMat);
body.position.y = 1.2;
playerGroup.add(body);

// 頭（視覚的区別用）
const headGeo = new THREE.SphereGeometry(0.3, 8, 8);
const headMat = new THREE.MeshStandardMaterial({ color: 0xffcc99 });
const head = new THREE.Mesh(headGeo, headMat);
head.position.y = 2.1;
playerGroup.add(head);

// 武器（右手に保持）
const gunGeo = new THREE.BoxGeometry(0.15, 0.15, 0.6);
const gunMat = new THREE.MeshStandardMaterial({ color: 0x333333 });
const gun = new THREE.Mesh(gunGeo, gunMat);
gun.position.set(0.5, 1.4, -0.3);
playerGroup.add(gun);

scene.add(playerGroup);

// カメラ基準の相対移動（TPSの核心）
function updatePlayer(delta) {
  const speed = 6;
  
  // カメラの前方ベクトルを取得（Y成分を0＝水平面のみ）
  const forward = new THREE.Vector3();
  camera.getWorldDirection(forward);
  forward.y = 0;
  forward.normalize();
  
  // 右ベクトル（前方×上）
  const right = new THREE.Vector3();
  right.crossVectors(forward, new THREE.Vector3(0, 1, 0)).normalize();
  
  // ジョイスティック入力から移動方向を計算
  const moveDir = new THREE.Vector3();
  if (joystickForward) moveDir.add(forward);
  if (joystickBack) moveDir.sub(forward);
  if (joystickRight) moveDir.add(right);
  if (joystickLeft) moveDir.sub(right);
  
  if (moveDir.length() > 0) {
    moveDir.normalize();
    playerGroup.position.addScaledVector(moveDir, speed * delta);
    
    // 移動方向を向く
    const targetPos = playerGroup.position.clone().add(moveDir);
    playerGroup.lookAt(targetPos.x, playerGroup.position.y, targetPos.z);
  }
  
  // 地面衝突（簡易: y=0固定）
  playerGroup.position.y = 0;
}
```

### 調査結果
- **CapsuleGeometry** がThree.js標準のキャラクター形状。Boxより自然
- **カメラ基準移動**がTPSの最重要パターン。`camera.getWorldDirection()` で前方ベクトル取得、Y=0で水平化
- 移動方向に `lookAt()` でキャラクターを回転

---

## C. カメラシステム ★最重要要素★

### 要件
- Spring Arm方式（キャラクター背後から追従）
- タッチスワイプでカメラ回転（Orbit）
- 壁との衝突回避（Raycast）
- ズーム機能（射撃時）
- 滑らかな追従（Exponential Damping / lerp）

### 最適実現方法
```javascript
// Spring Armパラメータ
const SPRING_ARM = {
  DISTANCE: 8,        // キャラからカメラまでの距離
  HEIGHT: 3.5,        // カメラの高さオフセット
  LERP_FACTOR: 0.1,   // 追従の滑らかさ（0〜1）
  MIN_PITCH: -0.5,    // 上下角度の下限（ラジアン）
  MAX_PITCH: 1.2,     // 上下角度の上限
  ZOOM_DISTANCE: 4,   // ADS時のズーム距離
};

// カメラ回転状態
let cameraYaw = 0;      // 左右回転（Y軸）
let cameraPitch = 0.3;  // 上下回転（X軸）
let currentZoom = SPRING_ARM.DISTANCE;
let targetZoom = SPRING_ARM.DISTANCE;

function updateCamera(delta) {
  // ズームの滑らかな遷移
  currentZoom += (targetZoom - currentZoom) * 0.1;
  
  // カメラ目標位置を計算（球座標→直交座標）
  const spherical = new THREE.Spherical(
    currentZoom,
    cameraPitch,
    cameraYaw
  );
  
  const camOffset = new THREE.Vector3().setFromSpherical(spherical);
  const targetPos = playerGroup.position.clone().add(camOffset);
  targetPos.y += SPRING_ARM.HEIGHT;
  
  // Raycastによる衝突回避（カメラとキャラクターの間に障害物があるか）
  const raycaster = new THREE.Raycaster(
    targetPos.clone(),           // カメラ位置から
    playerGroup.position.clone().sub(targetPos).normalize(), // キャラ向けて
    0,                           // near
    currentZoom                  // far
  );
  
  // 障害物との交差をチェック（簡易版）
  const intersects = raycaster.intersectObjects(scene.children, true);
  let finalCamPos = targetPos;
  
  for (const hit of intersects) {
    if (hit.distance < currentZoom && hit.point.y > playerGroup.position.y) {
      // キャラとカメラの間に障害物 → カメラを障害物の手前に移動
      finalCamPos = raycaster.ray.at(hit.distance - 0.5, new THREE.Vector3());
      break;
    }
  }
  
  // lerpで滑らかに追従（Spring Arm効果）
  camera.position.lerp(finalCamPos, SPRING_ARM.LERP_FACTOR);
  camera.lookAt(playerGroup.position.clone().add(new THREE.Vector3(0, 1.5, 0)));
}

// タッチスワイプでカメラ回転
let lastTouchX = 0, lastTouchY = 0;
const TOUCH_SENSITIVITY = 0.005;

canvas.addEventListener('touchstart', (e) => {
  // 右側画面のタッチのみカメラ回転として処理
  const touch = e.touches[0];
  if (touch.clientX > window.innerWidth / 2) {
    lastTouchX = touch.clientX;
    lastTouchY = touch.clientY;
  }
}, { passive: true });

canvas.addEventListener('touchmove', (e) => {
  const touch = e.touches[0];
  if (touch.clientX > window.innerWidth / 2) {
    const dx = touch.clientX - lastTouchX;
    const dy = touch.clientY - lastTouchY;
    
    cameraYaw += dx * TOUCH_SENSITIVITY;
    cameraPitch = Math.max(SPRING_ARM.MIN_PITCH, 
              Math.min(SPRING_ARM.MAX_PITCH, cameraPitch + dy * TOUCH_SENSITIVITY));
    
    lastTouchX = touch.clientX;
    lastTouchY = touch.clientY;
  }
}, { passive: true });

// ズーム（ADS時）
function startAiming() { targetZoom = SPRING_ARM.ZOOM_DISTANCE; }
function endAiming() { targetZoom = SPRING_ARM.DISTANCE; }
```

### 調査結果
- **Spring Arm** はTPSのデファクトスタンダード。Unity/Unreal両方で標準サポート
- `THREE.Spherical` で球座標からカメラ位置を計算（実装が最もシンプル）
- **Raycast衝突回避** が必須。カメラが壁に埋まらないようにする
- `lerp()` によるExponential Dampingで滑らかな追従（硬すぎず、遅すぎない）
- **yomotsu/camera-controls** ライブラリも候補だが、依存増なので自前実装推奨

---

## D. タッチ操作

### 要件
- バーチャルジョイスティック（左側 — 移動）
- 射撃ボタン（右側上 — タップで発射）
- カメラ回転スワイプ（右側キャンバス領域）
- リロード/カバボタン（右側下）
- マルチタッチ対応（同時に操作可能）

### 最適実現方法 — nipplejs

```html
<!-- nipplejs CDN -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/nipplejs/0.10.1/nipplejs.min.js"></script>

<!-- UIコンテナ -->
<div id="ui-layer" style="position:fixed;top:0;left:0;width:100%;height:100%;pointer-events:none;z-index:10;">
  <!-- ジョイスティックゾーン（左下） -->
  <div id="joystick-zone" style="position:absolute;bottom:20px;left:20px;width:150px;height:150px;pointer-events:auto;"></div>
  
  <!-- 射撃ボタン -->
  <button id="fire-btn" style="position:absolute;bottom:80px;right:30px;width:70px;height:70px;border-radius:50%;background:rgba(255,68,68,0.7);border:3px solid white;color:white;font-size:14px;font-weight:bold;pointer-events:auto;">FIRE</button>
  
  <!-- リロードボタン -->
  <button id="reload-btn" style="position:absolute;bottom:20px;right:110px;width:55px;height:55px;border-radius:50%;background:rgba(68,136,255,0.7);border:2px solid white;color:white;font-size:11px;font-weight:bold;pointer-events:auto;">R</button>
  
  <!-- カバボタン -->
  <button id="cover-btn" style="position:absolute;bottom:20px;right:30px;width:55px;height:55px;border-radius:50%;background:rgba(68,255,68,0.7);border:2px solid white;color:white;font-size:11px;font-weight:bold;pointer-events:auto;">COVER</button>
</div>
```

```javascript
// nipplejsジョイスティック設定
const joystickManager = nipplejs.create({
  zone: document.getElementById('joystick-zone'),
  mode: 'static',           // 固定位置
  position: { left: '50%', top: '50%' },
  color: 'white',
  size: 120,                // ジョイスティックの大きさ
});

// ジョイスティック入力状態
const joystickInput = { x: 0, y: 0, forward: false, back: false, left: false, right: false };

joystickManager.on('move', (evt, data) => {
  if (data && data.vector) {
    joystickInput.x = data.vector.x;
    joystickInput.y = data.vector.y;
    
    // 方向フラグ（閾値付き）
    const threshold = 0.3;
    joystickInput.forward = data.vector.y > threshold;
    joystickInput.back = data.vector.y < -threshold;
    joystickInput.right = data.vector.x > threshold;
    joystickInput.left = data.vector.x < -threshold;
  }
});

joystickManager.on('end', () => {
  joystickInput.x = 0;
  joystickInput.y = 0;
  joystickInput.forward = false;
  joystickInput.back = false;
  joystickInput.right = false;
  joystickInput.left = false;
});

// 射撃ボタン（Pointer Events — マルチタッチ対応）
let isFiring = false;
const fireBtn = document.getElementById('fire-btn');

fireBtn.addEventListener('pointerdown', (e) => {
  e.preventDefault();
  isFiring = true;
  shoot();
}, { passive: false });

fireBtn.addEventListener('pointerup', (e) => {
  isFiring = false;
});

// リロードボタン
document.getElementById('reload-btn').addEventListener('click', () => reload());

// カバボタン（長押し検知）
let coverTimer = null;
const coverBtn = document.getElementById('cover-btn');

coverBtn.addEventListener('pointerdown', (e) => {
  e.preventDefault();
  startCover();
  coverTimer = setTimeout(() => {}, 200); // 簡易: タップでもカバ
});

coverBtn.addEventListener('pointerup', () => endCover());
```

### 調査結果
- **nipplejs** がモバイルジョイスティックのデファクトスタンダード。CDN経由で利用可能
- `mode: 'static'` で固定位置配置（ドラッグで移動しない）
- **Pointer Events API** をボタンのタッチに使用（マルチタッチ対応）。`touchstart` だと指が入れ替わるとバグる
- ジョイスティックの `data.vector` から x/y 方向を取得。閾値付きで方向フラグ化

---

## E. 射撃システム

### 要件
- Hit Scan（即座命中判定）または Projectile（物理弾道）
- リコイルエフェクト（カメラ揺れ）
- マズルフラッシュ
- 弾数管理 + リロード
- ヒットマーカー / 命中エフェクト

### 最適実現方法 — Hybrid方式（Hit Scan + 簡易Projectile視覚）

```javascript
// 射撃状態
const WEAPON = {
  AMMO_MAX: 30,
  AMMO_CURRENT: 30,
  FIRE_RATE: 0.15,       // 連射間隔（秒）
  RECOIL_AMOUNT: 0.05,   // リコイル量
  RECOVERY_SPEED: 2.0,   // リコイル回復速度
  RELOAD_TIME: 2.0,      // リロード時間
};

let lastFireTime = 0;
let recoilOffset = 0;
let isReloading = false;

function shoot() {
  if (isReloading || WEAPON.AMMO_CURRENT <= 0) return;
  
  const now = performance.now() / 1000;
  if (now - lastFireTime < WEAPON.FIRE_RATE) return;
  lastFireTime = now;
  
  WEAPON.AMMO_CURRENT--;
  
  // Hit Scan: カメラの前方へレイを飛ばす
  const raycaster = new THREE.Raycaster();
  raycaster.setFromCamera(new THREE.Vector2(0, 0), camera); // 画面中央
  
  // 敵との交差チェック
  const enemyMeshes = STATE.enemies.map(e => e.mesh);
  const intersects = raycaster.intersectObjects(enemyMeshes, true);
  
  if (intersects.length > 0 && intersects[0].distance < 100) {
    // 命中！敵のHPを減少
    const hitObject = intersects[0].object;
    let enemy = STATE.enemies.find(e => e.mesh === hitObject || e.mesh.children.includes(hitObject));
    if (enemy) {
      enemy.takeDamage(25);
      createHitEffect(intersects[0].point);
    }
  }
  
  // リコイル（カメラを上にずらす）
  recoilOffset += WEAPON.RECOIL_AMOUNT;
  
  // マズルフラッシュ
  createMuzzleFlash();
  
  // プロジェクトイルの視覚エフェクト（簡易線）
  createTracer(intersects[0] ? intersects[0].point : raycaster.ray.at(50, new THREE.Vector3()));
}

function createHitEffect(position) {
  // 命中位置にパーティクルを生成
  const particleCount = 8;
  const positions = new Float32Array(particleCount * 3);
  const colors = new Float32Array(particleCount * 4);
  
  for (let i = 0; i < particleCount; i++) {
    positions[i*3] = position.x;
    positions[i*3+1] = position.y;
    positions[i*3+2] = position.z;
    
    colors[i*4] = 1; colors[i*4+1] = 0.8; colors[i*4+2] = 0; colors[i*4+3] = 1;
  }
  
  const geo = new THREE.BufferGeometry();
  geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
  geo.setAttribute('color', new THREE.BufferAttribute(colors, 4));
  
  const mat = new THREE.PointsMaterial({ size: 0.2, vertexColors: true });
  const particles = new THREE.Points(geo, mat);
  scene.add(particles);
  
  // 簡易アニメーション（frame後に削除）
  setTimeout(() => scene.remove(particles), 200);
}

function createMuzzleFlash() {
  const flash = new THREE.PointLight(0xffaa44, 10, 8);
  flash.position.copy(camera.position);
  const forward = new THREE.Vector3();
  camera.getWorldDirection(forward);
  flash.position.addScaledVector(forward, -1);
  scene.add(flash);
  setTimeout(() => scene.remove(flash), 50);
}

function createTracer(target) {
  // カメラ位置から命中点へ線を描画（簡易トレーサー）
  const points = [camera.position.clone(), target];
  const geo = new THREE.BufferGeometry().setFromPoints(points);
  const mat = new THREE.LineBasicMaterial({ color: 0xffff44, transparent: true, opacity: 0.6 });
  const line = new THREE.Line(geo, mat);
  scene.add(line);
  setTimeout(() => scene.remove(line), 100);
}

function reload() {
  if (isReloading || WEAPON.AMMO_CURRENT === WEAPON.AMMO_MAX) return;
  isReloading = true;
  
  // リロード中、HUDにアニメーション表示
  
  setTimeout(() => {
    WEAPON.AMMO_CURRENT = WEAPON.AMMO_MAX;
    isReloading = false;
  }, WEAPON.RELOAD_TIME * 1000);
}

// リコイル更新（毎フレーム）
function updateRecoil(delta) {
  if (recoilOffset > 0) {
    recoilOffset -= WEAPON.RECOVERY_SPEED * delta;
    if (recoilOffset < 0) recoilOffset = 0;
  }
}
```

### 調査結果
- **Hit Scan** が実装シンプル。Raycasterで画面中央からレイを飛ばすだけ
- **Projectile** は視覚エフェクト用（線/球体）に使い、判定はHit Scanと併用がベスト
- リコイルはカメラのpitchに加算する方式が簡単
- マズルフラッシュは `PointLight` を一瞬表示するだけ

---

## F. 敵AI — ステートマシン

### 要件
- パトロール → 追跡 → 攻撃 の状態遷移
- プレイヤーとの距離・視界で状態判定
- 攻撃クールダウン
- HPシステム + 死亡処理

### 最適実現方法

```javascript
class Enemy {
  constructor(position, type = 'basic') {
    this.type = type;
    this.health = type === 'basic' ? 50 : 100;
    this.maxHealth = this.health;
    this.speed = type === 'basic' ? 3 : 5;
    this.damage = type === 'basic' ? 10 : 20;
    this.detectionRange = 20;   // 感知範囲
    this.attackRange = 3;       // 攻撃範囲
    this.patrolRadius = 15;     // パトロール半径
    
    // メッシュ生成
    const geo = new THREE.CapsuleGeometry(0.4, 1.2, 4, 8);
    const color = type === 'basic' ? 0xe74c3c : 0x9b59b6;
    const mat = new THREE.MeshStandardMaterial({ color });
    this.mesh = new THREE.Mesh(geo, mat);
    this.mesh.position.copy(position);
    this.mesh.castShadow = true;
    scene.add(this.mesh);
    
    // HPバー（メッシュ上部に追加）
    this.healthBar = this.createHealthBar();
    
    // AI状態
    this.state = 'patrol';
    this.patrolTarget = this.randomPatrolPoint();
    this.attackCooldown = 0;
    this.alertTimer = 0;
  }
  
  createHealthBar() {
    const barGeo = new THREE.PlaneGeometry(1, 0.1);
    const barMat = new THREE.MeshBasicMaterial({ color: 0x00ff00, side: THREE.DoubleSide });
    const bar = new THREE.Mesh(barGeo, barMat);
    bar.position.y = 2.8;
    this.mesh.add(bar);
    return bar;
  }
  
  randomPatrolPoint() {
    return new THREE.Vector3(
      this.mesh.position.x + (Math.random() - 0.5) * this.patrolRadius * 2,
      0,
      this.mesh.position.z + (Math.random() - 0.5) * this.patrolRadius * 2
    );
  }
  
  // プレイヤーの視界内かチェック
  canSeePlayer() {
    const dist = this.mesh.position.distanceTo(playerGroup.position);
    if (dist > this.detectionRange) return false;
    
    // 方向チェック（プレイヤーが前方にいるか）
    const toPlayer = new THREE.Vector3().subVectors(playerGroup.position, this.mesh.position).normalize();
    const forward = new THREE.Vector3();
    this.mesh.getWorldDirection(forward);
    forward.y = 0; forward.normalize();
    
    const dot = forward.dot(toPlayer);
    return dot > 0.5; // 前方60度以内
  }
  
  update(delta) {
    // クールダウン減少
    this.attackCooldown -= delta;
    
    // HPバー更新
    this.updateHealthBar();
    
    const distToPlayer = this.mesh.position.distanceTo(playerGroup.position);
    
    // 状態遷移
    switch (this.state) {
      case 'patrol':
        if (this.canSeePlayer()) {
          this.state = 'chase';
        } else {
          this.moveTo(this.patrolTarget, delta);
          if (this.mesh.position.distanceTo(this.patrolTarget) < 1) {
            this.patrolTarget = this.randomPatrolPoint();
          }
        }
        break;
        
      case 'chase':
        if (!this.canSeePlayer() && distToPlayer > this.detectionRange * 1.5) {
          this.state = 'patrol';
        } else if (distToPlayer <= this.attackRange) {
          this.state = 'attack';
        } else {
          // プレイヤーへ向かって移動
          const dir = new THREE.Vector3().subVectors(playerGroup.position, this.mesh.position).normalize();
          dir.y = 0;
          this.mesh.position.addScaledVector(dir, this.speed * delta);
          this.mesh.lookAt(new THREE.Vector3(
            playerGroup.position.x, this.mesh.position.y, playerGroup.position.z
          ));
        }
        break;
        
      case 'attack':
        if (distToPlayer > this.attackRange * 1.5) {
          this.state = 'chase';
        } else if (this.attackCooldown <= 0) {
          // プレイヤーにダメージ
          STATE.playerHealth -= this.damage;
          updateHUD();
          createHitEffect(playerGroup.position.clone().add(new THREE.Vector3(0, 1.5, 0)));
          this.attackCooldown = type === 'basic' ? 1.5 : 1.0;
        }
        break;
    }
    
    // 地面固定
    this.mesh.position.y = 0;
  }
  
  moveTo(target, delta) {
    const dir = new THREE.Vector3().subVectors(target, this.mesh.position).normalize();
    dir.y = 0;
    this.mesh.position.addScaledVector(dir, 2 * delta);
    this.mesh.lookAt(new THREE.Vector3(target.x, this.mesh.position.y, target.z));
  }
  
  updateHealthBar() {
    const ratio = Math.max(0, this.health / this.maxHealth);
    this.healthBar.scale.x = ratio;
    
    // HPが低いほど赤に
    if (ratio > 0.5) this.healthBar.material.color.setHex(0x00ff00);
    else if (ratio > 0.25) this.healthBar.material.color.setHex(0xffaa00);
    else this.healthBar.material.color.setHex(0xff0000);
  }
  
  takeDamage(amount) {
    this.health -= amount;
    
    // 被弾フラッシュ
    this.mesh.material.emissive.setHex(0xffffff);
    setTimeout(() => this.mesh.material.emissive.setHex(0x000000), 100);
    
    if (this.health <= 0) {
      this.die();
    }
  }
  
  die() {
    scene.remove(this.mesh);
    const idx = STATE.enemies.indexOf(this);
    if (idx > -1) STATE.enemies.splice(idx, 1);
    STATE.score += 100;
    updateHUD();
    
    // 死亡エフェクト（簡易: 縮小アニメーション）
    this.mesh.scale.set(0.01, 0.01, 0.01);
    scene.add(this.mesh);
  }
}

// 敵のスポーン
function spawnEnemy() {
  const angle = Math.random() * Math.PI * 2;
  const dist = 25 + Math.random() * 15; // プレイヤーから25〜40単位離して配置
  const pos = new THREE.Vector3(
    playerGroup.position.x + Math.cos(angle) * dist,
    0,
    playerGroup.position.z + Math.sin(angle) * dist
  );
  
  const enemy = new Enemy(pos);
  STATE.enemies.push(enemy);
}
```

### 調査結果
- **FSM（有限ステートマシン）** が敵AIの標準パターン。patrol→chase→attackの状態遷移
- **視界判定** は「距離 + dot product（前方角度）」で実装。Raycastはコスト高いので簡易版で十分
- **Godot/UnityのBest Practice** を参考: 感知範囲（大）と攻撃範囲（小）を分離
- HPバーは `PlaneGeometry` でメッシュに追加（常にカメラに向けるのは後で実装）

---

## G. 衝突・物理システム

### 要件
- プレイヤー vs 障害物（壁・箱）
- プロジェクトイル vs 敵
- カバ判定（掩体の後ろにいるか）
- 軽量実装（Three.js標準のRaycasterは重い）

### 最適実現方法 — 距離ベース + AABB

```javascript
// 障害物データ管理
const obstacles = [];

function createObstacle(x, z, w, h, d, color) {
  const geo = new THREE.BoxGeometry(w, h, d);
  const mat = new THREE.MeshStandardMaterial({ color });
  const mesh = new THREE.Mesh(geo, mat);
  mesh.position.set(x, h / 2, z);
  mesh.castShadow = true;
  mesh.receiveShadow = true;
  scene.add(mesh);
  
  // AABBデータ（衝突判定用）
  obstacles.push({
    mesh: mesh,
    minX: x - w/2, maxX: x + w/2,
    minY: 0, maxY: h,
    minZ: z - d/2, maxZ: z + d/2,
    radius: Math.max(w, d) / 2 // 円形衝突判定用
  });
}

// プレイヤー vs 障害物（距離ベース — 簡易かつ高速）
function checkPlayerObstacleCollision() {
  const playerRadius = 0.5;
  
  for (const obs of obstacles) {
    // XZ平面での円形衝突判定
    const closestX = Math.max(obs.minX, Math.min(playerGroup.position.x, obs.maxX));
    const closestZ = Math.max(obs.minZ, Math.min(playerGroup.position.z, obs.maxZ));
    
    const dx = playerGroup.position.x - closestX;
    const dz = playerGroup.position.z - closestZ;
    const distSq = dx * dx + dz * dz;
    
    if (distSq < playerRadius * playerRadius && distSq > 0) {
      // 押し戻し
      const dist = Math.sqrt(distSq);
      const pushOut = (playerRadius - dist) / dist;
      playerGroup.position.x += dx * pushOut;
      playerGroup.position.z += dz * pushOut;
    }
  }
}

// プロジェクトイル vs 敵（距離ベース）
function checkProjectileEnemyCollision() {
  for (let i = STATE.projectiles.length - 1; i >= 0; i--) {
    const p = STATE.projectiles[i];
    
    for (const enemy of STATE.enemies) {
      if (p.position.distanceTo(enemy.mesh.position) < 1.5) {
        enemy.takeDamage(25);
        
        // 弾を削除
        scene.remove(p);
        STATE.projectiles.splice(i, 1);
        createHitEffect(p.position);
        break;
      }
    }
  }
}

// カバ判定（掩体の後ろにいるか）
function isInCover() {
  // プレイヤーからカメラ方向へRaycast
  const raycaster = new THREE.Raycaster();
  const camDir = new THREE.Vector3();
  camera.getWorldDirection(camDir);
  
  raycaster.set(playerGroup.position.clone().add(new THREE.Vector3(0, 1.5, 0)), camDir);
  const intersects = raycaster.intersectObjects(scene.children, true);
  
  for (const hit of intersects) {
    if (hit.distance < 5 && hit.point.y > 0.5) { // 5単位以内の障害物
      return true;
    }
  }
  return false;
}
```

### 調査結果
- **距離ベース判定** がThree.jsでは最も軽量。`distanceTo()` で十分
- AABBは障害物のmin/max座標を事前に計算しておけば高速
- カバ判定はRaycasterでカメラ方向をチェック（簡易版）
- BVH（Bounding Volume Hierarchy）も候補だが、依存増なので距離ベース推奨

---

## H. ゲームシステム

### 要件
- スコア管理 + ウェーブ制スパニング
- HPシステム + ゲームオーバー
- リロードタイマー
- 弾数表示

### 最適実現方法

```javascript
const STATE = {
  playerHealth: 100,
  maxHealth: 100,
  score: 0,
  wave: 1,
  enemiesRemaining: 0,
  isPlaying: false,
  isGameOver: false,
  enemies: [],
  projectiles: [],
};

// ウェーブ制スパニング
let spawnTimer = 0;
const SPAWN_INTERVAL = 3; // 秒

function updateSpawning(delta) {
  if (!STATE.isPlaying || STATE.isGameOver) return;
  
  spawnTimer += delta;
  if (spawnTimer >= SPAWN_INTERVAL && STATE.enemies.length < 5 + STATE.wave * 2) {
    spawnEnemy();
    spawnTimer = 0;
    
    // 全敵撃破後に次のウェーブ
    if (STATE.enemies.length > 0 && STATE.enemies.every(e => e.health <= 0)) {
      STATE.wave++;
    }
  }
}

// ゲームオーバー判定
function checkGameOver() {
  if (STATE.playerHealth <= 0 && !STATE.isGameOver) {
    STATE.isGameOver = true;
    showGameOver();
  }
}

function restartGame() {
  STATE.playerHealth = STATE.maxHealth;
  STATE.score = 0;
  STATE.wave = 1;
  STATE.enemies.forEach(e => scene.remove(e.mesh));
  STATE.enemies = [];
  STATE.isGameOver = false;
  WEAPON.AMMO_CURRENT = WEAPON.AMMO_MAX;
  
  document.getElementById('game-over-screen').style.display = 'none';
  updateHUD();
}
```

---

## I. UI/HUD — レスポンシブオーバーレイ

### 要件
- HPバー、スコア、弾数表示
- タッチボタン（ジョイスティック以外）
- レスポンシブ（画面サイズ自動調整）
- ゲームオーバー/スタート画面

### 最適実現方法 — HTML/CSSオーバーレイ

```html
<!-- HUDオーバーレイ -->
<div id="hud" style="position:fixed;top:0;left:0;width:100%;height:100%;pointer-events:none;z-index:10;">
  
  <!-- 上部HUD -->
  <div style="position:absolute;top:15px;left:15px;right:15px;display:flex;justify-content:space-between;align-items:center;">
    <!-- HPバー -->
    <div id="hp-container" style="width:200px;height:24px;background:rgba(0,0,0,0.6);border:2px solid #fff;border-radius:4px;overflow:hidden;">
      <div id="hp-fill" style="height:100%;background:#e74c3c;width:100%;transition:width 0.3s;"></div>
    </div>
    
    <!-- スコア -->
    <div id="score-display" style="color:white;font-size:20px;font-weight:bold;text-shadow:2px 2px 4px rgba(0,0,0,0.8);">
      Score: 0
    </div>
    
    <!-- ウェーブ -->
    <div id="wave-display" style="color:white;font-size:16px;text-shadow:2px 2px 4px rgba(0,0,0,0.8);">
      Wave: 1
    </div>
  </div>
  
  <!-- 弾数表示（右下） -->
  <div id="ammo-display" style="position:absolute;bottom:90px;right:30px;color:white;font-size:24px;font-weight:bold;text-shadow:2px 2px 4px rgba(0,0,0,0.8);">
    30 / 30
  </div>
  
  <!-- スタート画面 -->
  <div id="start-screen" style="position:absolute;top:0;left:0;width:100%;height:100%;background:rgba(0,0,0,0.85);display:flex;align-items:center;justify-content:center;z-index:20;">
    <div style="text-align:center;color:white;font-family:sans-serif;padding:30px;">
      <h1 style="font-size:clamp(24px, 6vw, 48px);margin-bottom:20px;">🎮 BROWSER TPS</h1>
      <p style="font-size:clamp(12px, 3vw, 18px);line-height:1.8;margin-bottom:30px;">
        🕹️ 左ジョイスティック: 移動<br>
        👆 右スワイプ: カメラ回転<br>
        🔴 FIREボタン: 射撃<br>
        🟡 Rボタン: リロード<br>
        🟢 COVERボタン: カバ
      </p>
      <button id="start-btn" style="padding:15px 40px;font-size:clamp(16px, 4vw, 24px);background:#e74c3c;color:white;border:none;cursor:pointer;border-radius:8px;font-weight:bold;">START GAME</button>
    </div>
  </div>
  
  <!-- ゲームオーバー画面 -->
  <div id="game-over-screen" style="display:none;position:absolute;top:0;left:0;width:100%;height:100%;background:rgba(0,0,0,0.85);align-items:center;justify-content:center;z-index:20;">
    <div style="text-align:center;color:white;font-family:sans-serif;padding:30px;">
      <h1 style="font-size:clamp(24px, 6vw, 48px);color:#e74c3c;margin-bottom:20px;">GAME OVER</h1>
      <p id="final-score" style="font-size:clamp(16px, 4vw, 24px);margin:20px 0;"></p>
      <button onclick="restartGame()" style="padding:15px 40px;font-size:clamp(16px, 4vw, 24px);background:#e74c3c;color:white;border:none;cursor:pointer;border-radius:8px;font-weight:bold;">RESTART</button>
    </div>
  </div>
</div>

<script>
function updateHUD() {
  // HPバー
  const hpRatio = Math.max(0, STATE.playerHealth / STATE.maxHealth) * 100;
  document.getElementById('hp-fill').style.width = hpRatio + '%';
  
  // スコア
  document.getElementById('score-display').textContent = 'Score: ' + STATE.score;
  
  // ウェーブ
  document.getElementById('wave-display').textContent = 'Wave: ' + STATE.wave;
  
  // 弾数
  document.getElementById('ammo-display').textContent = 
    WEAPON.AMMO_CURRENT + ' / ' + WEAPON.AMMO_MAX;
}

function showGameOver() {
  document.getElementById('final-score').textContent = 'Final Score: ' + STATE.score;
  document.getElementById('game-over-screen').style.display = 'flex';
}
</script>
```

### 調査結果
- **HTML/CSSオーバーレイ** がThree.jsキャンバスの上に重なる方式が標準
- `pointer-events:none` でHUD全体を透過、ボタン部分のみ `pointer-events:auto`
- `clamp()` CSS関数でレスポンシブフォントサイズ（スマホ/PC両対応）
- nipplejsのジョイスティックは別DOM要素（`position: absolute`）

---

## J. パフォーマンス最適化

### 要件
- モバイルで30fps以上維持
- LOD（遠くは低ポリゴン）
- InstancedMesh（弾・パーティクル共通化）
- シェード数制限

### 最適実現方法

```javascript
// デバイス判定
const isMobile = /Android|iPhone|iPad/i.test(navigator.userAgent);

// レンダラー設定
const renderer = new THREE.WebGLRenderer({ antialias: !isMobile });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // モバイルは最大2
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = isMobile ? THREE.BasicShadowMap : THREE.PCFSoftShadowMap;

// InstancedMesh — プロジェクトイル共通化
const MAX_BULLETS = 50;
const bulletGeo = new THREE.SphereGeometry(0.1, 6, 6);
const bulletMat = new THREE.MeshBasicMaterial({ color: 0xffdd44 });
const instancedBullets = new THREE.InstancedMesh(bulletGeo, bulletMat, MAX_BULLETS);
instancedBullets.count = 0;
scene.add(instancedBullets);

// InstancedMesh更新（毎フレーム）
function updateInstancedProjectiles() {
  let count = 0;
  const matrix = new THREE.Matrix4();
  
  for (const p of STATE.projectiles) {
    if (count >= MAX_BULLETS) break;
    matrix.setPosition(p.position.x, p.position.y, p.position.z);
    instancedBullets.setMatrixAt(count++, matrix);
  }
  instancedBullets.count = count;
  instancedBullets.instanceMatrix.needsUpdate = true;
}

// LOD — 遠くの障害物は低ポリゴン版に切り替え
function createLODObstacle(x, z, w, h, d) {
  const lod = new THREE.LOD();
  
  // 高詳細（近い）
  const highGeo = new THREE.BoxGeometry(w, h, d);
  const mat = new THREE.MeshStandardMaterial({ color: 0x5c5c7a });
  lod.addLevel(new THREE.Mesh(highGeo, mat), 0);
  
  // 低詳細（遠い）
  const lowGeo = new THREE.BoxGeometry(w, h, d);
  lod.addLevel(new THREE.Mesh(lowGeo, mat), 30);
  
  lod.position.set(x, h/2, z);
  scene.add(lod);
}

// フレームレート監視
let frameCount = 0, lastFPSTime = performance.now();
function monitorFPS() {
  const now = performance.now();
  frameCount++;
  if (now - lastFPSTime >= 1000) {
    console.log('FPS:', frameCount);
    
    // 30fps未満なら自動で品質低下
    if (frameCount < 30 && !isMobile) {
      renderer.shadowMap.enabled = false;
      console.log('⚠️ Low FPS — shadows disabled');
    }
    
    frameCount = 0;
    lastFPSTime = now;
  }
}

// オブジェクトプーリング（敵の生成/削除を効率化）
const enemyPool = [];
function getEnemyFromPool() {
  for (let i = 0; i < enemyPool.length; i++) {
    if (!enemyPool[i].active) return enemyPool[i];
  }
  // 新しい敵を作成
  const e = new Enemy(new THREE.Vector3());
  e.active = true;
  enemyPool.push(e);
  return e;
}

function recycleEnemy(enemy) {
  enemy.active = false;
  scene.remove(enemy.mesh);
}
```

### 調査結果（100 Tips from utsubo.com 2026）
- **WebGPU renderer** が2026年の新標準。Three.js r167+で実験的サポート
- **Draco/KTX2** アセット圧縮が必須（モバイル向け）
- **InstancedMesh** は10個以上の重複オブジェクトで必須
- `setPixelRatio` を2に制限するのがモバイル最適化の第一歩
- FPS監視して30未満なら自動品質低下が実用的

---

## K. 環境・レベルデザイン

### 要件
- プロシージャルマップ生成（毎回変わる）
- カバ用の障害物配置
- ライティング（雰囲気演出）
- フォグ（奥行き表現）

### 最適実現方法

```javascript
function generateArena() {
  // 地面
  const groundGeo = new THREE.PlaneGeometry(80, 80);
  const groundMat = new THREE.MeshStandardMaterial({ color: 0x3a3a5c });
  const ground = new THREE.Mesh(groundGeo, groundMat);
  ground.rotation.x = -Math.PI / 2;
  ground.receiveShadow = true;
  scene.add(ground);
  
  // グリッドヘルパー（視覚的ガイド）
  const grid = new THREE.GridHelper(80, 20, 0x4a4a6a, 0x3a3a5c);
  scene.add(grid);
  
  // ランダム障害物（掩体として機能）
  for (let i = 0; i < 12; i++) {
    const w = 2 + Math.random() * 4;
    const d = 2 + Math.random() * 4;
    const h = 1.5 + Math.random() * 2;
    
    // プレイヤーの近くに生成しない（距離チェック）
    let x, z;
    do {
      x = (Math.random() - 0.5) * 60;
      z = (Math.random() - 0.5) * 60;
    } while (Math.sqrt(x*x + z*z) < 10); // プレイヤーから10単位以内は避ける
    
    createObstacle(x, z, w, h, d, 0x5c5c7a);
  }
  
  // 柱（垂直障害物）
  for (let i = 0; i < 6; i++) {
    const pillarGeo = new THREE.CylinderGeometry(0.8, 0.8, 5, 12);
    const pillarMat = new THREE.MeshStandardMaterial({ color: 0x4a4a6a });
    const pillar = new THREE.Mesh(pillarGeo, pillarMat);
    
    let x, z;
    do {
      x = (Math.random() - 0.5) * 55;
      z = (Math.random() - 0.5) * 55;
    } while (Math.sqrt(x*x + z*z) < 8);
    
    pillar.position.set(x, 2.5, z);
    pillar.castShadow = true;
    scene.add(pillar);
    obstacles.push({
      mesh: pillar, radius: 0.8,
      minX: x-0.8, maxX: x+0.8, minZ: z-0.8, maxZ: z+0.8, minY: 0, maxY: 5
    });
  }
  
  // ライティング
  const ambient = new THREE.AmbientLight(0x2a2a4a, 0.4);
  scene.add(ambient);
  
  const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
  dirLight.position.set(15, 25, 10);
  dirLight.castShadow = true;
  dirLight.shadow.mapSize.width = isMobile ? 1024 : 2048; // モバイルは低解像度
  dirLight.shadow.mapSize.height = isMobile ? 1024 : 2048;
  scene.add(dirLight);
  
  // フォグ（奥行き表現 + レンダリング最適化）
  scene.fog = new THREE.FogExp2(0x1a1a2e, 0.015);
}
```

---

## 📊 要素優先順位マトリクス

| 優先度 | 要素 | 理由 | フェーズ |
|-------|------|------|---------|
| **P0** | C. カメラシステム | TPSの命。これが悪いとゲームにならない | Phase 1 |
| **P0** | B. キャラクター移動 | 基本操作。必須 | Phase 1 |
| **P0** | D. タッチ操作 | モバイルTPSの入力基盤 | Phase 1 |
| **P1** | A. コアレンダリング | シーン描画の土台 | Phase 1 |
| **P1** | E. 射撃システム | TPSの中核機能 | Phase 2 |
| **P1** | F. 敵AI | ゲームプレイの相手 | Phase 3 |
| **P2** | G. 衝突・物理 | 基本碰撞は必要だが高度な物理は不要 | Phase 1-2 |
| **P2** | H. ゲームシステム | スコア/HPでゲームらしくする | Phase 4 |
| **P2** | I. UI/HUD | 情報表示は必須 | Phase 4 |
| **P3** | K. 環境・レベル | 見た目のクオリティ | Phase 5 |
| **P3** | J. パフォーマンス | 最適化は後からで良い | Phase 6 |

---

## 🔧 新規作成すべきスキル一覧

| スキル名 | 内容 | 既存スキルとの関係 |
|---------|------|------------------|
| `browser-tps-mobile` | nipplejs + Spring Armカメラ + マルチタッチUIの統合パターン | threejs-3d-action-game + threejs-racing-game のTPS特化版 |
| `threejs-raycast-shooting` | Hit Scan射撃 + プロジェクトイル視覚エフェクト | threejs-3d-action-game の射撃部分を詳細化 |
| `threejs-enemy-fsm` | 敵AIステートマシン（patrol/chase/attack）パターン | threejs-3d-action-game の敵AIを詳細化 |

---

*調査日: 2025-09-09*
*情報ソース: Three.js公式, discourse.threejs.org, nipplejs GitHub, utsubo.com, gamecn.dev, 複数チュートリアル*
