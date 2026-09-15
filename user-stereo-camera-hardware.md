---
name: user-stereo-camera-hardware
description: 使用者的雙目相機是淘寶 1MP UVC 免驅同步模組，無出廠標定、無深度晶片、無 IMU。
metadata:
  type: project
---

> ⚠ **2026-08-28 擱置**：改用已有的 2D 光達，見 [[lidar-ogm-forecast-spec-v2]]。以下僅存歷史脈絡。

使用者為 [[cclstm-dynamic-static-project]] 買的相機：淘寶「100萬像素雙目同步VR測距攝像頭3D深度檢測模組高清USB免驅模組」。

實際上這是**標準 UVC 雙鏡頭模組**，不是深度相機：
- 免驅（UVC）→ OpenCV `VideoCapture` 直接讀，left/right 以 side-by-side 單張影像輸出
- **雙目硬體同步**（這點對 stereo 很重要，選得對）
- 「3D深度檢測」是行銷詞，**板上沒有深度晶片**，disparity/depth 全部要自己算
- **沒有出廠標定** → 內參、外參、baseline 都要自己用棋盤格標定 + rectify
- **沒有 IMU** → 要 ego-motion 得自己做 VO/VIO

實用範圍推估（2×640×480、HFOV≈60°、f≈554px、baseline≈6cm，**規格需實測確認**）：
`Z = fB/d ≈ 33.2/d` 公尺。d=32px→1.0m(±1.5cm)、16px→2.1m(±6cm)、8px→4.2m(±24cm)、4px→8.3m(±96cm)。
**可用深度大約 0.5–4 m，室內用。**

**How to apply:** 這個範圍跟 [[cclstm-paper-spec]] 的 100×100 m 駕駛場景差 25 倍，任何沿用論文超參數（320 格、0.3125 m/cell、預測 8 秒）的做法都不成立，必須縮尺。
