---
name: bev-system-spec-v1
description: ⚠ 已被 lidar-ogm-forecast-spec-v2 取代（改用 2D 光達）。仍保留有價值的 BEV 幾何教訓。
metadata:
  type: project
---

> ⚠ **2026-08-28 已被 [[lidar-ogm-forecast-spec-v2]] 取代**——感測器改為 2D 光達，雙目相機方案擱置。
> 本檔仍值得保留：分箱／格心公式、raycast 可見性、自監督 GT 這幾項在光達版一樣適用。
> 標定、SGBM、高度切片、`r_near` 近場死區已作廢（相機特有）。

2026-08-27 使用者拍板：**做一套自己的即時系統，CCLSTM 只是借用的架構**（不是重現論文數字，不用 WOMD/AV2）。

**規格（相對於 [[cclstm-paper-spec]] 縮尺）：**
- BEV 格：**128×128 @ 4 cm** = 5.12×5.12 m（128/4=32，符合 encoder 4 倍降採樣）
- 座標：ego-centric，相機在下緣中央，X∈[−2.56,2.56]、Z∈[0,5.12]
- 歷史 T_h=10 @10 Hz (1 s)；預測 **T_f=10 @10 Hz (1 s)**，跑通後再延到 2 s
- **高度切片 0.7–1.4 m 的軀幹帶，相機架 1.05 m、俯角 0**（見下方近場死區）
- 輸入 channel：**occupancy(1) + visibility(1) + static prior(1) + flow(2)**
- 潛在維度 **C=64，約 2M 參數**（論文 C=256/31M）

**三個關鍵取代：**
1. semantic map → **static prior map**（長時間累積的房間骨架：牆、家具）
2. 新增 **visibility mask channel**——相機只看得到扇形，必須區分「已知 free」與「未觀測」，否則網路把未觀測當 free 學
3. flow 由**使用者自己的動靜態模組**產生：BEV 連通元件 + 跨幀配對 → blob 位移 → rasterize 成 F∈ℝ^{H×W×2}。動靜態判斷換到正確座標系後就是 flow 產生器，不是另一件事。

**v1 相機固定在腳架上不動。** 這樣不需要 pose/VO/TF（原 pipeline 的 Pose Alignment 整段消失），動靜態退化成 BEV 背景相減，也不用 ego-motion channel。等 v1 跑通再讓相機動。

**訓練資料：未來 occupancy 的 GT 是免費的，就是自己錄影的後面幾幀**（自監督，零人工標註）。錄 30–60 分鐘室內有人走動，10 Hz → 18k–36k 幀。flow GT 才需要 blob tracking 弱標註；論文消融顯示沒有 flow 也能跑（Obs AUC 0.79→0.76），所以 **v1 先只用 L_occ**。

**階段：** 0 標定 → 1 SGBM depth（捲尺驗證）→ 2 BEV occupancy+visibility raycast → 3 動靜態+flow → 4 錄資料集 → 5 移植訓練 CCLSTM → 6 即時整合。延遲預估 SGBM 20ms + BEV 2ms + 推論 15ms ≈ 25 fps。

專案目錄：`/home/user/bev_forecast/`。相關：[[cclstm-dynamic-static-project]]、[[user-stereo-camera-hardware]]


---
**2026-08-27 階段 2 實作發現（重要，別再重推）：**

**1. 垂直 FOV 造成近場死區。** 相機在切片正中央、俯角 0 時：
`r_near = (h_max − h_min) / (2·tan(vFOV/2))`
近於 r_near，視錐涵蓋不到整個高度切片，該處**不能宣稱 free**（看不到 ≠ 沒有），否則假 free 會教壞網路。
640×480 / HFOV 60°（vFOV≈47°）下實測：切片 0.1–2.0 m → 死區 2.19 m（吃掉網格 43%）；**切片 0.7–1.4 m → 死區 0.81 m，可用深度 84%**。所以選軀幹帶。
**往下轉相機會讓死區變大**（向上的光線構不到 h_max），俯角 20° 時死區 16.8 m，30° 時無解。相機要水平。
低矮靜態障礙（桌腳）改由 static prior map 用另一組較寬切片離線累積，它不需要即時。

**2. BEV 分箱與格心必須用同一組公式**（踩過）：
`row = floor((z_max − Z_g)/res)`，格 i 涵蓋 `Z ∈ [z_max−(i+1)·res, z_max−i·res)`，格心 `= z_max−(i+0.5)·res`。
用 `S−1−z/res` 分箱卻用 `(S−0.5−i)·res` 當格心會差一格，表面落到格子外、被自己的可見性判定濾掉。

**3. 可見性用方位角 raycast**：只拿切片內的點算每個方位的最近距離 r_min（地板/天花板不該造成遮擋），格心 r ≤ r_min + 0.75·res 才算已觀測。這是 BEV 的 2D 近似，h_min 就是控制「能不能越過矮物看後面」的旋鈕。

**進度：階段 2 完成並通過合成場景驗證**（`tools/test_synthetic.py`，6 項幾何斷言全過）。
程式：`bev/config.py`、`bev/projection.py`（純 numpy）、`tools/probe_camera.py`（相機一插上就能跑）。
環境：OpenCV 4.6.0、numpy 1.26.4、**RTX 4060 Laptop 8 GB**、**torch 尚未安裝**（階段 5 再裝）。
相機尚未寄到：/dev/video0,1 都是筆電內建 Acer webcam。
