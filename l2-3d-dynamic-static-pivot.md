---
name: l2-3d-dynamic-static-pivot
description: 2026-10-19 轉向——感測器從 2D 光達（M10P）換成宇樹 Unitree L2 3D 光達，題目改成 3D 點雲動靜態分離＋底盤/手臂避讓分工；工作區 ~/l2_ws 有 CLAUDE.md
metadata: 
  node_type: memory
  type: project
  originSessionId: fd444371-fc80-4fb2-9886-b7574d5f1718
  modified: 2026-10-19T07:18:35.565Z
---

2026-10-19：專案換到 `~/l2_ws`，**規格細節一律看 `~/l2_ws/CLAUDE.md`（那是權威來源，不要從記憶背規格）**。

**與舊方向的差異**
- 感測器：2D 光達（LSLIDAR M10P）→ **宇樹 Unitree L2 3D 光達**，倒裝在柱頂。
  → [[lidar-ogm-forecast-spec-v2]] 的 2D 格子大小、取樣頻率、M10P 參數陷阱**不再適用於新題目**，
    但「角度裁剪／補償會偷偷毀掉遮擋結論」那類方法論教訓仍然有效，換到 3D 也要同樣查驅動實際回報。
- 題目：OGM 佔據預測 → **3D 點雲動靜態分離**，輸出物體位置＋速度給規劃器。
- 平台：仍是 WHEELTEC S100 差速底盤（見 [[wheeltec-s100-platform]]），但現在是**移動機械手臂**，
  研究問題落在「底盤與手臂的避讓分工與時機，以及預測誤差多大時好處消失」。
- 現階段全在 **Gazebo Harmonic 模擬**，不是實機錄資料。

**Why**：使用者 2026-10-19 直接給了新的 CLAUDE.md，內容與 9 月的 2D／OGM 規格不相容，
中間一個多月的決策過程未記錄。

**How to apply**：被問到這個專案時以 `~/l2_ws/CLAUDE.md` 為準；v2 記憶只當歷史脈絡與方法論
教訓引用，別拿它的數字當現行規格。[[forecast-baseline-comparison]] 的「先比 persistence
／等速外插」仍然適用——速度估計要跟等速外插比才知道值不值得。
