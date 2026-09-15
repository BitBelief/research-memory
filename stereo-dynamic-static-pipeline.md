---
name: stereo-dynamic-static-pipeline
description: 已選定的第一版 pipeline：Stereo → Depth → Voxel → Pose Alignment → 跨時間比較 → Dynamic/Static。
metadata:
  type: project
---

[[cclstm-dynamic-static-project]] 的**第一版 pipeline 已定案**為「中等難度」那條：

```
Stereo Camera → Depth → 3D Voxel → Pose Alignment → Temporal Occupancy Comparison → Dynamic / Static Voxel
```

**Why:** 曾評估過三種難度——(1) 兩幀 3D 點直接做 nearest-neighbor 距離比較、(2) voxel grid 跨時間比較 V_t-1 vs V_t、(3) Dynablox/nvblox free-space history。選 (2) 的理由：比 nvblox 簡單很多、不需要先建完整 TSDF/ESDF、而且輸出格式本來就是後面 CCLSTM 要的 Dynamic Occupancy、每一步都看得清楚。

**How to apply:** 討論這個專案時不要再把 nvblox 拉回主線，也不要在 nvblox / Dynablox / point cloud 之間跳來跳去——那正是使用者想擺脫的混亂。深度要**有真實尺度（metric）**，用 Z = fB/d（f 焦距、B baseline、d disparity），不要建議單目深度估計（scale ambiguity、domain shift、遠距離誤差）。實作順序是先把 Left/Right → Disparity → Depth 跑出來看到畫面，再往上疊。

注意：depth image 不等於把 3D 壓回 2D 丟掉高度——每個 pixel 存 Z，配合 (u,v,Z,K) 可完整還原 (X,Y,Z)。

---
**⚠ 2026-08-27 修正：** 讀過 [[cclstm-paper-spec]] 後發現終點對不上。CCLSTM 的輸入是 **2D BEV occupancy + flow 場**，不是 3D voxel 的 dynamic/static 二元標籤。上面這條 pipeline 的前段（stereo → depth → pose 對齊 → 跨時間比較）仍然有效，但輸出格式要改成 BEV raster，而且還缺 flow 那一路。方向待使用者確認：重現論文（用 WOMD/AV2，相機無關）vs 自建縮尺室內系統（相機有用，但要縮到 4×4m / 80×80 格 / 預測 ~1 秒）。
