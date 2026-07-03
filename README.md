# vrlab — 35 kV SST 微電網 Innovation Lab

一個以 **Three.js** 打造的 35 kV SST（固態變壓器）微電網實驗室 3D 展示場景，
可在桌機瀏覽器操作，也支援 **Meta Quest 系列頭盔的 WebXR 沉浸式體驗**，
用於對客戶／高層做實驗室導覽 demo。

## 專案內容

| 檔案 | 說明 |
|---|---|
| `index.html` | **主要版本**（單一 HTML，內嵌 ES Module JS）。桌機 3D 檢視 + 完整 WebXR／Quest 支援（傳送移動、控制器射線、空間 UI、設備剖面互動）。 |
| `lab_3d.html` | 較早期的桌機專用 3D 版本（OrbitControls，無 WebXR），做為 VR 改造前的基礎版本保留。 |
| `vr_dev_spec.md` | 給 Claude Code 的 WebXR 改造開發規格文件，記錄技術棧決策、改動清單與驗收標準（`index.html` 即依此規格實作完成）。 |

## 功能特色

- **場景**：以公尺為單位還原實驗室建築、9 個功能分區（Z1–Z9）、35 kV SST 集裝箱、
  BESS 儲能集裝箱、單軌吊車、屋頂與動線標示。
- **檢視模式**：Top（頂視）／ISO（等角）／Walk（第一人稱行走，WASD + 方向鍵）。
- **圖層切換**：屋頂顯示/半透明、工程動線、卡車動線、參觀動線。
- **設備剖面**：勾選後 SST／BESS 集裝箱外殼半透明化，顯示內部結構與浮動標籤。
- **WebXR / Meta Quest 支援**：
  - 偵測裝置支援度，顯示「Enter VR」按鈕，不支援則維持桌機操作。
  - 雙手控制器 + 射線指示，搭配 haptic 震動回饋。
  - 拋物線傳送移動，限制僅能傳送至參觀動線區域（高壓區禁止傳送）。
  - 左手腕浮現空間 3D UI 面板，可用右手射線點擊。
  - 目標效能：Quest 2 ≥ 72 fps／Quest 3 ≥ 90 fps。

## 分區配置（Zones）

| ID | 區域 | 說明 |
|---|---|---|
| Z1 | 展示前廳 | Innovation Showcase |
| Z2 | 中控室 | Control Room |
| Z3 | 參觀走廊 | Visitor Walkway |
| Z4 | 配電與電網模擬區 | Grid Simulation（高壓） |
| Z5 | SST 並聯測試區 | SST Parallel Test 35 kV（高壓） |
| Z6 | 直流微電網試驗區 | DC Microgrid, BESS + DC Bus（高壓） |
| Z7 | 工程操作區 | Engineering Workspace |
| Z8 | 檢修走廊 | Maintenance Aisle |
| Z9 | 卡車進出與吊裝通道 | Logistics & Lifting（高壓） |

## 技術棧

- [Three.js](https://threejs.org/) r160（透過 `importmap` 由 CDN 載入，無需建置流程）
- WebXR Device API（`VRButton`、`XRControllerModelFactory`）
- 純前端單一 HTML 檔案，無框架、無後端

## 本機執行

因為使用 ES Module 與 WebXR（需要 HTTPS 或 `localhost`），不能直接以 `file://` 開啟，
需透過本機伺服器：

```bash
# 任一方式皆可
python3 -m http.server 8000
# 或
npx serve .
```

然後瀏覽器開啟 `http://localhost:8000/index.html`。

若要在 Meta Quest 上以 WebXR 進入，需部署到 HTTPS 網址（例如 Vercel、Netlify），
在頭盔瀏覽器中開啟該網址並點擊「Enter VR」。

## 操作說明

**桌機**
- 快捷鍵：`1` Top、`2` ISO、`3` Walk
- Walk 模式：`WASD` 或方向鍵移動，滑鼠鎖定視角（左右方向鍵可轉向）
- 右側面板：切換視角、屋頂顯示、動線圖層、SST/BESS 剖面

**Meta Quest（WebXR）**
- 進入後站於參觀入口，左手腕面板顯示 View / Layers / Equipment 選單
- 右手射線瞄準地板 → 出現傳送圈 → 扣板機傳送（高壓區會顯示紅圈禁止傳送）
- 射線指向 SST／BESS 集裝箱並扣板機 → 外殼半透明化、顯示內部結構
