---
name: cclstm-paper-spec
description: CCLSTM 論文（arXiv 2506.06128）的實際輸入輸出規格——2D BEV，不是 3D voxel。
metadata:
  type: reference
---

**CCLSTM: Coupled Convolutional Long-Short Term Memory Network for Occupancy Flow Forecasting**
Peter Lengyel, aiMotive, arXiv:2506.06128v1 (2025-06-06). 本機 PDF：`/home/user/下載/2506.06128v1.pdf`
Code: https://github.com/aimotive/CCLSTM ／ https://aimotive.com/occupancy-forecasting

**表示法是 2D BEV raster，不是 3D voxel。**

輸入 X（WOMD）：occupancy grid `O_t ∈ ℝ^{H×W×1}`、semantic map `M ∈ ℝ^{H×W×3}`（道路拓撲+號誌畫成 RGB）、backward flow field `F ∈ ℝ^{H×W×2}`。
輸出 Y：future observed occupancy `H×W×1`、occluded occupancy `H×W×1`、backward flow `H×W×2`。
AV2 變體：occupancy + lane occupancy `L` + rasterized ego-motion flow `E`（IMU），ego-centric 座標。

尺寸：H,W=320 ↔ 100×100 m²（0.3125 m/cell）；挑戰賽用 512 ↔ 160×160 m²；評估中心 crop 256。
時間：歷史 T_h=10 @10Hz（過去 1s）→ 預測 T_f=8 @1Hz（未來 8s）。
架構：全卷積（3×3 與 5×5），31M 參數，encoder 4 層、空間降採樣 4 倍、embedding C=256。Accumulator CLSTM + Autoregressive Forecasting CLSTM，兩個 decoder 分支（occupancy / flow）。
Loss：`L = 1000·L_occ + 25·L_flow + 10·L_trace`；occupancy 用 reverse-flow 加權的 BCEWithLogits（α=10）補償資料集偏向靜止物。
訓練：WOMD 485,568 樣本，10 epochs，batch 32，單張 A100，AdamW lr 0.002。

**兩個關鍵陷阱：**
1. 輸入的 flow 由**帶 ID 的 agent 軌跡位移**推導——需要物件追蹤標註才有 GT。
2. 消融（Tab.3）證明 flow 輸入不能拿掉（Flow-EPE 2.60→3.15）；論文自承模型「may be suboptimal at estimating velocity from occupancy alone」。單靠 occupancy 不夠。

相關：[[cclstm-dynamic-static-project]]、[[stereo-dynamic-static-pipeline]]
