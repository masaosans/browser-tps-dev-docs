---
name: browser-tps-mobile
description: ブラウザTPS開発 — nipplejs, Spring Armカメラ, マルチタッチUI。スマホ向け3人称シューティング作成。
version: 1.0.0
author: Hermes Agent
license: MIT
---

# ブラウザTPS（モバイル特化）

Three.js + nipplejs でスマホブラウザ動作の3人称シューティングゲームを作成する。

## When to Use

- ユーザーが「ブラウザTPS」「スマホシューティング」「Three.js TPS」をリクエスト
- モバイルタッチ操作の3Dアクションゲームが必要
- バーチャルジョイスティック + Spring Armカメラの実装

## Stack

| Layer | Tool | Purpose |
|-------|------|---------|
| 3Dレンダリング | Three.js r162+ (CDN) | シーン描画、カメラ、ライティング |
| ジョイスティック | nipplejs 0.10.1 (CDN) | バーチャルアナログスティック |
| カメラ | Spring Arm + Spherical | TPS標準の追従カメラ |
| 衝突判定 | 距離ベース + AABB | 軽量・高速 |
| UI | HTML/CSSオーバーレイ | HUD + タッチボタン |

## Core Architecture

```javascript
// === ゲーム状態 ===
const STATE = {
  playerHealth: 100, score: 0, wave: 1, enemies: [],
  isPlaying: false, isGameOver: false,
};

// === Spring Armパラメータ ===
const SPRING_ARM = { DISTANCE: 8, HEIGHT: 3.5, LERP_FACTOR: 0.1, MIN_PITCH: -0.5, MAX_PITCH: 1.2 };

let cameraYaw = 0, cameraPitch = 0.3;
```

## Step 1: Three.jsシーン初期化（モバイル最適化）

```javascript
const isMobile = /Android|iPhone|iPad/i.test(navigator.userAgent);

const renderer = new THREE.WebGLRenderer({ antialias: !isMobile });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = isMobile ? THREE.BasicShadowMap : THREE.PCFSoftShadowMap;
document.body.appendChild(renderer.domElement);

window.addEventListener('resize', () => {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
});
```

## Step 2: プレイヤーキャラクター + カメラ基準移動

```javascript
const playerGroup = new THREE.Group();
const body = new THREE.Mesh(new THREE.CapsuleGeometry(0.4, 1.2, 4, 8),
  new THREE.MeshStandardMaterial({ color: 0x3498db }));
body.position.y = 1.2; playerGroup.add(body);

// カメラ基準の相対移動（TPSの核心パターン）
function updatePlayer(delta) {
  const speed = 6;
  const forward = new THREE.Vector3(); camera.getWorldDirection(forward);
  forward.y = 0; forward.normalize();
  const right = new THREE.Vector3().crossVectors(forward, new THREE.Vector3(0,1,0)).normalize();

  const moveDir = new THREE.Vector3();
  if (joystickForward) moveDir.add(forward);
  if (joystickBack) moveDir.sub(forward);
  if (joystickRight) moveDir.add(right);
  if (joystickLeft) moveDir.sub(right);

  if (moveDir.length() > 0) {
    moveDir.normalize();
    playerGroup.position.addScaledVector(moveDir, speed * delta);
    const targetPos = playerGroup.position.clone().add(moveDir);
    playerGroup.lookAt(targetPos.x, playerGroup.position.y, targetPos.z);
  }
  playerGroup.position.y = 0;
}
```

## Step 3: Spring Armカメラ + タッチスワイプ回転

```javascript
function updateCamera(delta) {
  currentZoom += (targetZoom - currentZoom) * 0.1;
  
  const spherical = new THREE.Spherical(currentZoom, cameraPitch, cameraYaw);
  const camOffset = new THREE.Vector3().setFromSpherical(spherical);
  const targetPos = playerGroup.position.clone().add(camOffset);
  targetPos.y += SPRING_ARM.HEIGHT;

  // Raycast衝突回避
  const raycaster = new THREE.Raycaster(targetPos, 
    playerGroup.position.clone().sub(targetPos).normalize(), 0, currentZoom);
  const intersects = raycaster.intersectObjects(scene.children, true);
  
  let finalCamPos = targetPos;
  for (const hit of intersects) {
    if (hit.distance < currentZoom && hit.point.y > playerGroup.position.y) {
      finalCamPos = raycaster.ray.at(hit.distance - 0.5, new THREE.Vector3());
      break;
    }
  }

  camera.position.lerp(finalCamPos, SPRING_ARM.LERP_FACTOR);
  camera.lookAt(playerGroup.position.clone().add(new THREE.Vector3(0, 1.5, 0)));
}

// タッチスワイプ（右側画面のみ）
const TOUCH_SENSITIVITY = 0.005;
let lastTouchX = 0, lastTouchY = 0;

canvas.addEventListener('touchstart', (e) => {
  const t = e.touches[0];
  if (t.clientX > window.innerWidth / 2) { lastTouchX = t.clientX; lastTouchY = t.clientY; }
}, { passive: true });

canvas.addEventListener('touchmove', (e) => {
  const t = e.touches[0];
  if (t.clientX > window.innerWidth / 2) {
    cameraYaw += (t.clientX - lastTouchX) * TOUCH_SENSITIVITY;
    cameraPitch = Math.max(SPRING_ARM.MIN_PITCH, Math.min(SPRING_ARM.MAX_PITCH, 
      cameraPitch + (t.clientY - lastTouchY) * TOUCH_SENSITIVITY));
    lastTouchX = t.clientX; lastTouchY = t.clientY;
  }
}, { passive: true });
```

## Step 4: nipplejsジョイスティック統合

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/nipplejs/0.10.1/nipplejs.min.js"></script>
<div id="ui-layer" style="position:fixed;top:0;left:0;width:100%;height:100%;pointer-events:none;z-index:10;">
  <div id="joystick-zone" style="position:absolute;bottom:20px;left:20px;width:150px;height:150px;pointer-events:auto;"></div>
  <button id="fire-btn" style="position:absolute;bottom:80px;right:30px;width:70px;height:70px;border-radius:50%;background:rgba(255,68,68,0.7);border:3px solid white;color:white;font-size:14px;font-weight:bold;pointer-events:auto;">FIRE</button>
  <button id="reload-btn" style="position:absolute;bottom:20px;right:110px;width:55px;height:55px;border-radius:50%;background:rgba(68,136,255,0.7);border:2px solid white;color:white;font-size:11px;font-weight:bold;pointer-events:auto;">R</button>
</div>
```

```javascript
const joystickManager = nipplejs.create({
  zone: document.getElementById('joystick-zone'), mode: 'static',
  position: { left: '50%', top: '50%' }, color: 'white', size: 120,
});

const joystickInput = { x: 0, y: 0, forward: false, back: false, left: false, right: false };

joystickManager.on('move', (evt, data) => {
  if (!data?.vector) return;
  const t = 0.3;
  joystickInput.x = data.vector.x; joystickInput.y = data.vector.y;
  joystickInput.forward = data.vector.y > t;
  joystickInput.back = data.vector.y < -t;
  joystickInput.right = data.vector.x > t;
  joystickInput.left = data.vector.x < -t;
});

joystickManager.on('end', () => {
  Object.assign(joystickInput, { x:0, y:0, forward:false, back:false, left:false, right:false });
});

// Pointer Events for buttons (multitouch-safe)
document.getElementById('fire-btn').addEventListener('pointerdown', (e) => { e.preventDefault(); shoot(); }, { passive: false });
document.getElementById('reload-btn').addEventListener('click', () => reload());
```

## Step 5: Hit Scan射撃 + エフェクト

```javascript
const WEAPON = { AMMO_MAX: 30, AMMO_CURRENT: 30, FIRE_RATE: 0.15, RECOIL_AMOUNT: 0.05 };
let lastFireTime = 0, recoilOffset = 0;

function shoot() {
  const now = performance.now()/1000;
  if (now - lastFireTime < WEAPON.FIRE_RATE || WEAPON.AMMO_CURRENT <= 0) return;
  lastFireTime = now; WEAPON.AMMO_CURRENT--;

  const rc = new THREE.Raycaster();
  rc.setFromCamera(new THREE.Vector2(0,0), camera);
  const hits = rc.intersectObjects(STATE.enemies.map(e => e.mesh), true);
  
  if (hits.length > 0 && hits[0].distance < 100) {
    let enemy = STATE.enemies.find(e => e.mesh === hits[0].object || e.mesh.children.includes(hits[0].object));
    if (enemy) { enemy.takeDamage(25); createHitEffect(hits[0].point); }
  }

  recoilOffset += WEAPON.RECOIL_AMOUNT;
  
  // マズルフラッシュ
  const flash = new THREE.PointLight(0xffaa44, 10, 8);
  flash.position.copy(camera.position);
  const fwd = new THREE.Vector3(); camera.getWorldDirection(fwd);
  flash.position.addScaledVector(fwd, -1); scene.add(flash);
  setTimeout(() => scene.remove(flash), 50);
}

function reload() {
  if (WEAPON.AMMO_CURRENT === WEAPON.AMMO_MAX) return;
  WEAPON.AMMO_CURRENT = WEAPON.AMMO_MAX; // 簡易:即時リロード
}
```

## Step 6: 敵AIステートマシン（最小限）

```javascript
class Enemy {
  constructor(pos, type='basic') {
    this.health = type==='basic' ? 50 : 100;
    this.speed = type==='basic' ? 3 : 5;
    this.damage = type==='basic' ? 10 : 20;
    this.detectionRange = 20; this.attackRange = 3;
    
    const m = new THREE.Mesh(new THREE.CapsuleGeometry(0.4,1.2,4,8),
      new THREE.MeshStandardMaterial({ color: type==='basic' ? 0xe74c3c : 0x9b59b6 }));
    m.position.copy(pos); scene.add(m); this.mesh = m;
    this.state = 'patrol'; this.attackCooldown = 0;
  }

  canSeePlayer() {
    const d = this.mesh.position.distanceTo(playerGroup.position);
    if (d > this.detectionRange) return false;
    const toP = new THREE.Vector3().subVectors(playerGroup.position, this.mesh.position).normalize();
    const fwd = new THREE.Vector3(); this.mesh.getWorldDirection(fwd); fwd.y=0; fwd.normalize();
    return fwd.dot(toP) > 0.5;
  }

  update(delta) {
    this.attackCooldown -= delta;
    const d = this.mesh.position.distanceTo(playerGroup.position);

    if (this.state === 'patrol' && this.canSeePlayer()) this.state = 'chase';
    else if (this.state === 'chase') {
      if (!this.canSeePlayer() && d > this.detectionRange*1.5) this.state = 'patrol';
      else if (d <= this.attackRange) this.state = 'attack';
      else {
        const dir = new THREE.Vector3().subVectors(playerGroup.position, this.mesh.position).normalize();
        dir.y=0; this.mesh.position.addScaledVector(dir, this.speed*delta);
        this.mesh.lookAt(new THREE.Vector3(playerGroup.position.x, 0, playerGroup.position.z));
      }
    } else if (this.state === 'attack') {
      if (d > this.attackRange*1.5) this.state = 'chase';
      else if (this.attackCooldown <= 0) { STATE.playerHealth -= this.damage; updateHUD(); this.attackCooldown = 1.5; }
    }
    this.mesh.position.y = 0;
  }

  takeDamage(amount) {
    this.health -= amount;
    this.mesh.material.emissive.setHex(0xffffff);
    setTimeout(() => this.mesh.material.emissive.setHex(0x000000), 100);
    if (this.health <= 0) {
      scene.remove(this.mesh);
      const i = STATE.enemies.indexOf(this); if(i>-1) STATE.enemies.splice(i,1);
      STATE.score += 100; updateHUD();
    }
  }
}
```

## Step 7: HUD + レスポンシブUI

```html
<div id="hud" style="position:fixed;top:0;left:0;width:100%;height:100%;pointer-events:none;z-index:10;">
  <div style="position:absolute;top:15px;left:15px;right:15px;display:flex;justify-content:space-between;">
    <div id="hp-container" style="width:200px;height:24px;background:rgba(0,0,0,0.6);border:2px solid #fff;border-radius:4px;overflow:hidden;">
      <div id="hp-fill" style="height:100%;background:#e74c3c;width:100%;transition:width 0.3s;"></div>
    </div>
    <div id="score-display" style="color:white;font-size:20px;font-weight:bold;text-shadow:2px 2px 4px rgba(0,0,0,0.8);">Score: 0</div>
  </div>
  <div id="ammo-display" style="position:absolute;bottom:90px;right:30px;color:white;font-size:24px;font-weight:bold;text-shadow:2px 2px 4px rgba(0,0,0,0.8);">30 / 30</div>
</div>

<script>
function updateHUD() {
  document.getElementById('hp-fill').style.width = (STATE.playerHealth/STATE.maxHealth*100)+'%';
  document.getElementById('score-display').textContent = 'Score: '+STATE.score;
  document.getElementById('ammo-display').textContent = WEAPON.AMMO_CURRENT+' / '+WEAPON.AMMO_MAX;
}
</script>
```

## Performance Checklist

- [ ] `setPixelRatio(Math.min(devicePixelRatio, 2))` — モバイルは最大2
- [ ] `antialias: !isMobile` — モバイルでOFF
- [ ] `BasicShadowMap` on mobile — PCのみPCFSoft
- [ ] `InstancedMesh` for bullets/particles (10+ objects)
- [ ] LOD for distant obstacles
- [ ] FPS monitor → auto quality drop if < 30fps

## Pitfalls & Gotchas

- **nipplejs container must have CSS position** — `position: absolute/relative/fixed` を設定しないと配置失敗。`static` だと警告
- **Pointer Events API for buttons** — `touchstart` だけだとマルチタッチで指が入れ替わるとバグる。`pointerdown/up/move` + `pointerId` チェックを使用
- **Camera rotation only on right side** — 左側はジョイスティックと干渉。`clientX > window.innerWidth / 2` で判定
- **Joystick threshold** — 0.3程度の閾値で方向フラグ化（ノイズ対策）
- **Raycast for camera collision** — カメラとキャラクターの間に障害物があるかチェック
- **Spherical coordinates for camera** — `THREE.Spherical` で球座標→直交座標変換が最もシンプル
- **Delta time for movement** — `clock.getDelta()` を使用。フレームレート非依存
- **Audio context requires user gesture** — Web Audioは初回クリック後に初期化

## References

- nipplejs: https://github.com/yoannmoinet/nipplejs
- Three.js TPS Tutorial: https://www.abratabia.com/threejs/third-person-shooter-tutorial.php
- Spring Arm Camera: https://discourse.threejs.org/t/third-person-camera/18624
- Mobile FPS Tips: https://www.utsubo.com/blog/threejs-best-practices-100-tips
