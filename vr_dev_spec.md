# Meta Quest VR 版開發規格 — 35 kV SST 微電網實驗室

> **任務**：把 `lab_3d.html`（既有 Three.js Web 版）轉成 Meta Quest 2／3／Pro 可進入並互動的 WebXR 版本，輸出檔名 `lab_vr.html`。
> **執行者**：Claude Code。
> **目標讀者**：George（技術 co-founder），用於對客戶／高層做沉浸式 demo。

---

## 1. 技術棧決策（已定，不要改）

| 決策點 | 選擇 | 理由 |
|---|---|---|
| 引擎 | **Three.js r160 + WebXR API** | 既有程式碼基礎；不需學 Unity；一份程式碼，桌機與 Quest 都能跑 |
| 部署 | **HTTPS 靜態網頁** | WebXR 強制 HTTPS；可放 Vercel／Netlify／自建 |
| 移動方式 | **Teleport（傳送）** | VR 標準移動，不會引發暈眩 |
| UI | **空間浮動面板**（不是 DOM） | DOM UI 在 VR 內看不到，必須換成 3D 面板 |
| 著色器 | 維持 `MeshStandardMaterial`，**移除** `MeshPhysicalMaterial.transmission` | transmission 在 Quest 上掉幀嚴重 |
| 陰影 | **關閉** `renderer.shadowMap.enabled` 或限制單一光源 | Quest 移動 GPU 吃不消 |
| 目標幀率 | **≥ 72 fps（Quest 2）/ 90 fps（Quest 3）** | 低於此會暈眩；用 Stats panel 監控 |

---

## 2. 改動清單（vs `lab_3d.html`）

從現有檔案出發，按以下順序改造。每一步都是獨立 commit。

### 2.1 啟用 WebXR

```js
import { VRButton } from 'three/addons/webxr/VRButton.js';
import { XRControllerModelFactory } from 'three/addons/webxr/XRControllerModelFactory.js';

renderer.xr.enabled = true;
document.body.appendChild(VRButton.createButton(renderer));

// 重要：改用 setAnimationLoop 取代 requestAnimationFrame
renderer.setAnimationLoop(animate);
```

### 2.2 場景縮放校正

VR 內 1 unit = 1 公尺，現有檔案就是公尺單位，**不需縮放**。但要注意：

- 起始相機位置改為房間入口、眼睛高度（Y=1.7）
- `camera.position.set(4, 1.7, 13.5)` → 站在參觀入口
- 移除 OrbitControls，VR 內由頭部追蹤接管

### 2.3 控制器與射線

加入兩支控制器、射線指示、抓取點：

```js
const controllerModelFactory = new XRControllerModelFactory();

for (let i = 0; i < 2; i++) {
  const controller = renderer.xr.getController(i);
  controller.addEventListener('selectstart', onSelectStart);
  controller.addEventListener('selectend', onSelectEnd);
  scene.add(controller);

  const grip = renderer.xr.getControllerGrip(i);
  grip.add(controllerModelFactory.createControllerModel(grip));
  scene.add(grip);

  // 射線（淡藍色發光線）
  const rayGeo = new THREE.BufferGeometry().setFromPoints([
    new THREE.Vector3(0,0,0), new THREE.Vector3(0,0,-1)
  ]);
  const ray = new THREE.Line(rayGeo, new THREE.LineBasicMaterial({ color: 0x00A6E5 }));
  ray.scale.z = 5;
  controller.add(ray);
}
```

### 2.4 傳送移動

實作以下行為：

1. 使用者按下「搖桿前推」或「Trigger」→ 從控制器發射拋物線
2. 拋物線終點打到地板 → 顯示藍色傳送圈（半徑 0.4 m）
3. 放開搖桿／Trigger → 把 `XRReferenceSpace` 平移到目標點

需限制傳送目標：

- **白名單區域**：Z1（前廳）、Z2（中控室）、Z3（參觀走廊）、Z8（檢修走廊邊緣 0.5 m 內）
- **禁止傳送**：Z4／Z5／Z6（高壓區）、Z9（卡車道）— 在玻璃隔斷外圍，傳送圈變紅色，無法落地

實作參考：https://threejs.org/examples/?q=teleport（webxr_vr_teleport）

### 2.5 空間 UI 面板

把現有 DOM 控制面板換成 3D 面板，黏在**左手控制器**上方（手腕內側）：

```
┌─────────────────────┐
│  視角 / View        │
│  [Top] [ISO] [出口] │   ← 不需 Walk，VR 本身就是 walk
├─────────────────────┤
│  圖層 / Layers      │
│  ☑ 屋頂  ☑ 玻璃     │
│  ☑ 工程  ☑ 卡車     │
│  ☑ 參觀  ☑ 行吊     │
├─────────────────────┤
│  設備 / Equipment   │
│  ☐ SST 剖面         │
│  ☐ BESS 剖面        │
└─────────────────────┘
```

實作：
- 用 `CanvasTexture` 在 `PlaneGeometry`（0.25m × 0.4m）上繪製 UI
- 面板綁在左控制器 grip 的 `children` 內，相對位置 `(0.05, 0.05, -0.1)`，旋轉 `(-Math.PI/4, 0, 0)`
- 右控制器射線指到面板上某個按鈕區域 → 高亮 → Trigger 觸發
- 點擊用 `Raycaster.intersectObject(panel)`，根據 `intersection.uv` 判斷點擊哪個按鈕

**互動回饋**：按鈕被指到時控制器震動（haptic）：
```js
controller.gamepad?.hapticActuators?.[0]?.pulse(0.3, 30);
```

### 2.6 設備互動（重點 demo 賣點）

當射線指到 **SST 集裝箱** 或 **BESS 集裝箱** 並按 Trigger：

- 集裝箱外殼半透明化（`opacity 0.22`）
- 內部結構顯現（變壓器三相線圈／電池模組陣列）
- 漂浮 3D 標籤（"5 MW Solid State Transformer / 35 kV"）
- 再按一次 → 還原

內部結構建議用簡化幾何（不是真實工業內部，而是「可辨識的視覺隱喻」）：
- SST 內部：3 個圓柱（變壓器線圈，紫色發光）+ 電力電子板（PCB 紋）
- BESS 內部：8 × 6 陣列的小方塊（電池模組，深綠色）+ 中央 BMS 控制盒

### 2.7 性能優化（必做，否則 Quest 上會很卡）

| 項目 | 原值 | 改為 | 理由 |
|---|---|---|---|
| `renderer.shadowMap.enabled` | true | **false** | 移動 GPU 吃不消 |
| `directionalLight.castShadow` | true | false | 同上 |
| `MeshPhysicalMaterial`（玻璃）| transmission 0.62 | 改用 `MeshStandardMaterial` + `opacity 0.3` | transmission 太重 |
| `renderer.toneMapping` | ACESFilmic | NoToneMapping | 多 1ms |
| `renderer.setPixelRatio` | min(devicePixelRatio, 2) | **WebXR 自動接管** | 不要手動設 |
| 行吊動畫 | sin wave | 維持 | 動畫成本可忽略 |
| 燈具陣列 | 多個 Mesh | **InstancedMesh** | 50+ 個重複 mesh 必須 instanced |
| Sprite labels | 雙語 1024×256 canvas | 簡化為 512×128 | 省記憶體 |

### 2.8 進入點與出口

- VR 啟動時：使用者站在參觀入口（X=4, Y=1.7, Z=13.5），面朝 SST 區
- 加入「Reset 位置」按鈕在 UI 面板上 — 萬一傳送出界

### 2.9 桌機相容性

加入 fallback：
- 偵測 `navigator.xr?.isSessionSupported('immersive-vr')`
- 不支援 → 回退到 OrbitControls + DOM UI（既有行為）
- 支援 → 顯示 "Enter VR" 按鈕

---

## 3. 檔案結構

```
project/
├── lab_vr.html         ← 主檔（單一 HTML，內嵌 JS）
├── README.md           ← 操作說明
└── assets/             ← 若需外部材質/字型（可選）
    └── notosans-tc.woff2
```

**保持單一 HTML**，所有 JS 內嵌，部署最簡單。

---

## 4. 互動腳本（給 Claude Code 的「人因」依據）

預期使用情境：

> **客戶經理帶外賓進入 VR**：
> 1. 戴上 Quest → 看到實驗室外觀，藍色 LED 標題浮現
> 2. 左手腕浮現 UI 面板
> 3. 右手射線指地板 → 出現傳送圈 → 按 Trigger → 瞬移到中控室
> 4. 從中控室玻璃幕牆看主實驗區，SST 集裝箱在運轉（行吊在動）
> 5. 切換 UI「屋頂」→ 屋頂淡出，可俯瞰
> 6. 走到參觀走廊，射線指 BESS 集裝箱 → Trigger → 集裝箱透明化，看到電池模組陣列
> 7. 整個 demo 5–8 分鐘

---

## 5. 驗收標準（Acceptance Criteria）

| # | 標準 | 量測方法 |
|---|---|---|
| 1 | Quest 2 上穩定 ≥ 72 fps | OVR Metrics Tool |
| 2 | Quest 3 上穩定 ≥ 90 fps | OVR Metrics Tool |
| 3 | 桌機 Chrome 仍正常顯示 | 視覺檢查 |
| 4 | 進 VR 後可成功傳送到 Z1/Z2/Z3 | 手動測試 |
| 5 | 嘗試傳送進 Z5（高壓區）→ 圈圈變紅，無法落地 | 手動測試 |
| 6 | 射線指 SST → Trigger → 變半透明 + 內部顯現 | 手動測試 |
| 7 | UI 面板按鈕點擊有 haptic 震動 | 手動測試 |
| 8 | 啟動到可互動 ≤ 5 秒 | 計時 |
| 9 | 不支援 WebXR 的瀏覽器仍可桌機操作 | Chrome 桌機測試 |

---

## 6. 不要做的事

- ❌ 不要改成 A-Frame／Babylon／PlayCanvas — 會丟失既有程式碼
- ❌ 不要加實時 GI／反射探針／後製 bloom — Quest 跑不動
- ❌ 不要把 DOM UI 留著當「電腦版控制台」— 會讓 VR 玩家以為控制器壞了
- ❌ 不要用平滑滑動移動（smooth locomotion）— 會暈，傳送是標準
- ❌ 不要綁 Quest 專屬 API（如手勢追蹤）— 影響桌機相容性

---

## 7. 給 Claude Code 的指令模板

複製下面這段話到 Claude Code：

```
我有一個 lab_3d.html（Three.js r160 ESM，35 kV SST 微電網實驗室 3D 場景）。
請依據附件 vr_dev_spec.md，將其改造成 Meta Quest 可進入的 WebXR 版本，輸出 lab_vr.html 到專案根目錄。

請按 §2 的順序逐項實作，每完成一項用 git commit 檔下來，commit message 用 §2 的小節標題。
完成後執行 §5 的驗收項目（你能自動檢查的）並回報。
```

附上 `lab_3d.html` 與本文件即可。
