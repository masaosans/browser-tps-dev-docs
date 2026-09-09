# 🎮 ブラウザTPSゲーム開発ロードマップ

> **目標:** スマホブラウザで動作する3人称シューティング（TPS）ゲーム
> **技術スタック:** Three.js + 単一HTMLファイル + CDN依存
> **プラットフォーム:** iOS Safari / Android Chrome（タッチ操作前提）

---

## 📋 全体構成図

```
┌─────────────────────────────────────────────────┐
│              index.html (単一ファイル)            │
├──────────────┬──────────────────────────────────┤
│   HTML/CSS   │  HUDオーバーレイ / タッチUI       │
│   JavaScript │  Three.jsシーン / ゲームロジック   │
├──────────────┼──────────────────────────────────┤
│  3Dレンダリング│  キャラクター / カメラ / 環境     │
│  物理演算    │  衝突判定 (簡易AABB/距離計算)      │
│  AIシステム  │  敵キャラクターの行動ステートマシン  │
│  射撃システム │  プロジェクトイル / ヒットスキャン   │
│  タッチ操作  │  バーチャルジョイスティック + ボタン │
└──────────────┴──────────────────────────────────┘
```

---

## 🛠️ 技術スタック決定

| レイヤー | ツール | 理由 |
|---------|--------|------|
| **3Dレンダリング** | Three.js r162+ (CDN) | ブラウザ標準、ドキュメント豊富 |
| **物理演算** | カスタム簡易衝突判定 | 軽量TPSには過剰。距離計算で十分 |
| **カメラ制御** | Spring Arm + Orbit | TPSの標準パターン（既存スキル参照） |
| **モバイル操作** | nipplejs (ジョイスティック) | 実績あるライブラリ、マルチタッチ対応 |
| **オーディオ** | Web Audio API (native) | 追加依存なし |
| **出力形式** | 単一HTMLファイル | ビルド不要、GitHub Pagesで即公開 |

---

## 📅 開発フェーズ（7段階）

### Phase 1: プロトタイプ — 基本移動 + カメラ ⭐最重要
**目標:** キャラクターが動いて、カメラが追従する最小構成

```
工程:
1. Three.jsシーン初期化（renderer, camera, scene, lights）
2. プレイヤーキャラクター作成（CapsuleGeometryで簡易人体）
3. Spring Armカメラ実装（キャラクター背後から追従）
4. タッチジョイスティックで移動可能に（nipplejs）
5. 画面タップでカメラ回転（右側スワイプ）
6. 地面と基本的な障害物を配置

成功基準:
✅ スマホブラウザでキャラクターが動く
✅ カメラがスムーズに追従する
✅ 壁にぶつかると停止する
```

**参考パターン:** `threejs-3d-action-game` スキルのTPSカメラ + `threejs-racing-game` のタッチ操作

---

### Phase 2: 射撃システム
**目標:** タップで弾を発射、敵に命中判定

```
工程:
1. 右側ボタンタップで射撃（Hit Scan方式）
2. プロジェクトイル（球体）の発射・移動・消去
3. ヒットマーカーエフェクト（画面中央フラッシュ）
4. マズルフラッシュ（ポイントライト）
5. 弾数管理 + リロード機能

成功基準:
✅ ボタンタップで弾が発射される
✅ プロジェクトイルが飛んでいく
✅ 命中エフェクトが表示される
```

**重要決定:** 初期はHit Scan（即座命中）→ 後にProjectile（物理弾道）へ拡張

---

### Phase 3: 敵AIシステム
**目標:** パトロール → 追跡 → 攻撃する敵キャラクター

```
工程:
1. Enemyクラス定義（ステートマシン: patrol/chase/attack）
2. NavMesh風経路探索（簡易版：プレイヤー方向へ直進）
3. 視界判定（Raycast風：距離 + 角度チェック）
4. 攻撃クールダウン + ダメージ処理
5. 敵の生死管理（HP0で削除、スコア加算）

成功基準:
✅ 敵がマップをパトロールする
✅ プレイヤーを見つけると追跡する
✅ 近づくと攻撃してくる
✅ 倒すとスコアに加算される
```

**AIステート遷移図:**
```
[Patrol] → (プレイヤーが視界内) → [Chase]
[Chase]  → (距離5以下)           → [Attack]
[Attack] → (クールダウン終了)     → [Chase]
[Chase]  → (プレイヤーが遠すぎる) → [Patrol]
```

---

### Phase 4: ゲームループ + UI
**目標:** スコア・HP表示、ゲームオーバー判定、リスタート

```
工程:
1. HUD実装（HPバー、スコア、弾数表示）
2. ウェーブ制スパニング（敵を段階的に追加）
3. プレイヤー被ダメージ処理
4. ゲームオーバー画面 + リスタート機能
5. スタート画面（操作説明付き）

成功基準:
✅ HPが0になるとゲームオーバー
✅ スコアが表示される
✅ リスタートで最初からやり直せる
```

---

### Phase 5: レベルデザイン + カバシステム
**目標:**掩体（カバー）のある戦場、戦略的プレイ

```
工程:
1. 障害物配置（壁、箱、柱 — 掩体として機能）
2. カバシステム（ボタン長押しで掩体の後ろに隠れる）
3. プロシージャルマップ生成（ランダム配置）
4. ライティング改善（ポイントライト + フォグ）

成功基準:
✅ プレイヤーが掩体の後ろに隠れることができる
✅ 敵も掩体を利用する（AI拡張）
✅ マップが毎回変わる（プロシージャル）
```

---

### Phase 6: ポリッシュ + パフォーマンス最適化
**目標:** 滑らかな動作、視覚的なクオリティ向上

```
工程:
1. モバイル向け最適化
   - LOD（遠くは低ポリゴン）
   - InstancedMesh（弾・パーティクル共通化）
   - アンチエイリアスOFF（モバイル時）
   - テクスチャ圧縮
2. パーティクルエフェクト
   - 命中時の火花
   - 爆発エフェクト
   - ダスト（足音）
3. サウンド効果
   - 射撃SE、被ダメージSE、BGM
4. アニメーション改善
   - キャラクターの歩きアニメーション
   - 弾丸ヒット時のフラッシュ

成功基準:
✅ スマホで60fps維持
✅ エフェクトが美しく見える
✅ サウンドが鳴る
```

---

### Phase 7: GitHub Pages公開 + フィードバック反映
**目標:** 世界中に公開、ユーザーフィードバック収集

```
工程:
1. リポジトリ作成（GitHub）
2. index.htmlをアップロード
3. GitHub Pages有効化
4. スマホで動作テスト（複数デバイス）
5. フィードバックに基づく改善

成功基準:
✅ URLで誰でもプレイ可能
✅ iOS/Android両方で動作確認済み
```

---

## 📱 モバイルUIレイアウト設計

```
┌─────────────────────────────────────┐
│  [スコア表示]        [HPバー]       │ ← 上部HUD
│                                    │
│         Three.js Canvas            │ ← メインゲーム画面
│         (3Dレンダリング)            │
│                                    │
│    ┌──────────┐   ┌───────┐        │
│    │          │   │  🔫   │        │ ← 右側: 射撃ボタン
│    │  🕹️      │   └───────┘        │
│    │ ジョイ    │   ┌───────┐        │
│    │ スティック │   │reload │        │ ← 右下: リロード
│    │ (移動)    │   └───────┘        │
│    │          │   ┌───────┐        │
│    └──────────┘   │ cover │        │ ← 右下: カバ
│                   └───────┘        │
└─────────────────────────────────────┘

左側: バーチャルジョイスティック（移動）
右側上: 射撃ボタン
右側下: リロード + カバボタン
```

**タッチ操作の詳細:**
- **左側ジョイスティック:** nipplejsライブラリで実装。ドラッグ方向にキャラクター移動
- **右側スワイプ:** キャンバス上のタッチ位置を記録、ドラッグ量でカメラ回転
- **射撃ボタン:** タップで即座に発射（連打可能）
- **リロードボタン:** 弾数0またはタップでリロード開始
- **カバボタン:** 長押しで掩体の後ろに移動

---

## 🧠 作成すべきスキル一覧

### 既存スキルの活用
| スキル | 活用箇所 |
|--------|---------|
| `threejs-3d-action-game` | TPSカメラ、射撃、敵AI、HUDの基本パターン |
| `threejs-racing-game` | モバイルタッチ操作（Pointer Events API）、パフォーマンス最適化 |

### 新規作成すべきスキル
1. **`browser-tps-mobile`** — ブラウザTPS × モバイル特化スキル
   - nipplejsジョイスティック統合パターン
   - マルチタッチUIレイアウト設計
   - Spring Armカメラ + タッチ回転の組み合わせ
   - モバイルパフォーマンス最適化チェックリスト

2. **`threejs-game-deployment`** — Three.jsゲームのGitHub Pages公開手順
   - リポジトリ作成 → ファイルアップロード → Pages有効化
   - モバイルテスト手順

---

## ⚡ パフォーマンス最適化チェックリスト（モバイル）

```javascript
// 1. デバイス判定
const isMobile = /Android|iPhone|iPad|iPod/i.test(navigator.userAgent);

// 2. レンダラー設定
renderer = new THREE.WebGLRenderer({ antialias: !isMobile });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2)); // モバイルは2まで

// 3. シェード数制限
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap; // または THREE.BasicShadowMap（高速）

// 4. InstancedMesh活用（10個以上の重複オブジェクト）
const bulletGeo = new THREE.SphereGeometry(0.1, 6, 6);
const bulletMat = new THREE.MeshBasicMaterial({ color: 0xffdd44 });
const instancedBullets = new THREE.InstancedMesh(bulletGeo, bulletMat, 50);

// 5. LOD（Level of Detail）
const lod = new THREE.LOD();
lod.addLevel(highPolyMesh, 0);       // 近い
lod.addLevel(mediumPolyMesh, 20);    // 中距離
lod.addLevel(lowPolyMesh, 50);       // 遠い

// 6. フレームレート監視
let frameCount = 0, lastTime = performance.now();
function checkFPS() {
  const now = performance.now();
  frameCount++;
  if (now - lastTime >= 1000) {
    console.log('FPS:', frameCount);
    frameCount = 0;
    lastTime = now;
  }
}

// 7. モバイル目標: 30fps以上を維持
```

---

## 🎯 優先順位（MVPファースト）

```
🥇 MVP (最低限動くもの): Phase 1-4
   - 移動 + カメラ + 射撃 + 敵AI + HUD
   - これだけで「ゲームとして成立」する

🥈 V1.0: Phase 5-6
   - カバシステム + パフォーマンス最適化 + エフェクト
   - 「楽しくなる」レベル

🥉 V2.0: Phase 7
   - GitHub Pages公開 + フィードバック反映
   - 「世界中に届ける」
```

---

## 🔗 参考リソース

| リソース | URL |
|---------|-----|
| Three.js TPSチュートリアル | https://www.abratabia.com/threejs/third-person-shooter-tutorial.php |
| BasicThirdPersonGame (GitHub) | https://github.com/matthias-schuetz/THREE-BasicThirdPersonGame |
| nipplejs (ジョイスティックライブラリ) | https://github.com/yoannmoinet/nipplejs |
| Three.jsモバイル最適化100選 | https://www.utsubo.com/blog/threejs-best-practices-100-tips |
| ブラウザ3Dゲーム完全ガイド | https://www.seeles.ai/resources/blogs/three-js-games-ultimate-guide |

---

## 📊 開発スケジュール目安

| フェーズ | 期間 | 難易度 |
|---------|------|--------|
| Phase 1: プロトタイプ | 2-3時間 | ⭐⭐ |
| Phase 2: 射撃システム | 2-3時間 | ⭐⭐ |
| Phase 3: 敵AI | 3-4時間 | ⭐⭐⭐ |
| Phase 4: ゲームループ+UI | 2-3時間 | ⭐⭐ |
| Phase 5: レベルデザイン | 3-4時間 | ⭐⭐⭐ |
| Phase 6: ポリッシュ | 4-5時間 | ⭐⭐⭐ |
| Phase 7: 公開 | 1-2時間 | ⭐ |

**合計目安: 17-24時間（約3-5日）**

---

*作成日: 2025-09-09*
*最新情報に基づいて作成。Three.jsやモバイルブラウザの仕様変更には注意。*
