---
name: cclstm-dynamic-static-project
description: 使用者的研究專案：用雙目相機做 3D voxel 動靜態判斷，最後接 CCLSTM；不是要做 3D 重建。
metadata:
  type: project
---

> ⚠ **2026-08-28 更新**：感測器已改為 **2D 光達**（雙目相機擱置），研究主軸已改為「校準且緊緻的可能範圍」。現行規格見 [[lidar-ogm-forecast-spec-v2]]。

使用者在做一個研究專案：從**雙目（stereo）RGB 相機**輸入，判斷空間中每個 voxel 是 **Dynamic 還是 Static**，輸出最後要餵給 **CCLSTM**（做 voxel/occupancy forecasting）。

關鍵：**目標不是 3D 重建**。使用者明確澄清過「我並沒有要 3D 重建，現在只是要判斷動靜態」。所以 nvblox / Dynablox 那套完整 TSDF/ESDF volumetric history 在現階段被判定為**非必要**，已從流程中拿掉。

nvblox 只有在採用 Dynablox 式邏輯（「這個位置長期是 Free，現在突然 Occupied」→ dynamic candidate）時才需要，因為那需要跨時間保存 free/occupied 的 3D 記憶。現階段不走這條。

硬體：使用者手上有一顆雙目相機。

來源：2026-08-27 從一份 ChatGPT 分享對話（標題「細看CCLSTM論文」）匯入。相關：[[stereo-dynamic-static-pipeline]]
