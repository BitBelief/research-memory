---
name: occupancy-forecast-prior-art-2026-08
description: 2026-08-28 文獻+程式碼查證結果——SCOPE/Occ3D/Bosch 各佔了什麼，以及唯一乾淨的缺口在哪。
metadata: 
  node_type: memory
  type: reference
  originSessionId: d5a2aa35-3a6f-43a8-ad6e-7e6ef9fff25b
  modified: 2026-08-30T14:40:12.743Z
---

2026-08-28 查證。**結論：三態不是缺口（自駕車的標配），校準與涵蓋率才是（三個領域全部 0）。**

## Xie & Dames（Temple Univ）——最接近的競爭者
- **SOGMP** CoRL 2023 / **SCOPE** IEEE T-RO 2025，DOI `10.1109/TRO.2025.3578234`
- 2D 光達 → OGM → VAE → 「一族可能的未來」+ uncertainty-aware planner，真機、開源
- code: `github.com/TempleRAIL/SOGMP`（含預訓練 `model/model.pth`）、`github.com/TempleRAIL/scope`、`scope_nav`
- 資料集 **OGM-datasets** on Zenodo `10.5281/zenodo.7051560`：Turtlebot2(模擬大廳,34 行人) / Jackal(SCAND,室外) / Spot(SCAND,室內)
- ⚠ README 寫 torch 1.7.1，**RTX 4060 跑不動**，要裝新版自己修相容

**程式碼實測（分支 `scope` / `scope++` / `so-scope`，`--depth 1` 只會拿到預設分支）：**
- 訓練目標與輸入都是 `discretize()` → **只把光達回波點標 1，其餘全 0**。沒有光線追蹤。
  → 網路學的是「**這格會不會有回波**」，不是「會不會被佔據」。`0` 同時代表 free／未觀測／被遮擋／障礙物內部。
- `update()` + `to_prob_occ_map()`（log-odds + Bresenham）**只在 ++ 分支用來建靜態地圖 m**，且 `to_prob_occ_map(0.8)` 會把 `P_prior=0.5` 的未觀測格二值化成 0=free。
- 參數：`P_prior=0.5, P_occ=0.7, P_free=0.3, RESOLUTION=0.1m, TRESHOLD_P_OCC=0.8`
- 損失：`BCELoss(sum) + BETA*KL`，**沒有任何項鼓勵校準**

**論文實測（發表版全文關鍵字）：** uncertainty 128 次，**calibration/coverage/conformal 各 0**，`discretiz` **0**（論文從未描述這一步）。
- 熵的定義明說假設「cells **independent**」且「occupied **or** free」的 **Bernoulli** → **公式層面就排除第三態，也拿不到聯合/軌跡級保證**
- 送規劃器前：「first **binarize** the predicted OGM and its uncertainty map using an occupancy threshold」→ 常數代價 → 高斯化。**不確定性的數值在此消失，門檻無機率意義**
- 自承：「this costmap-based framework **loses the stochastic consistency** of the entire navigation system」
- 遮擋 = **失敗案例 2**（Fig. 6b），且明寫「**will be our future work**」，打算用「更多粒子 / 更長歷史」去猜穿遮擋（= 等級 2）
- 真機只取 **8 個樣本**（算力限制），最低掉到 2 FPS → SO-SCOPE 用查表繞過取樣

## 自駕車那邊：三態是標配，不是缺口
- **Occ3D**（CVPR 2023 佔據預測挑戰賽資料集）：raycast 標 free，**射線上第一個佔據體素之後全部標 `unobserved`**，另有**相機可見性遮罩**
- **Bosch/Saarland, arXiv 2405.10575**（2024）用證據理論改善該 GT 品質：把 missing/occlusion/conflict 指派給「I don't know」假設；nuScenes GT 改善 30–52% MAE、下游預測再改善 25%。code: `github.com/boschresearch/evidential-occupancy`
- 但該篇 **calibration/coverage/conformal/guarantee/reliability 全部 0**
- ⚠ **任務不同**：Occ3D 那條是「從影像估計**當下**場景」（空間補全）；SCOPE/台大/你是「從過去 OGM 預測**未來**」（時序）。可講但別當主軸。

## 遮擋策略：兩邊都有，但旋鈕都沒刻度（2026-08-29 補查）
- **自駕車**：**phantom agent** —— 在遮擋區假設看不見的行人，算前向可達集合，規劃時避開；還會 **creep**（探頭獲取視野）。多篇：[Occlusion-Aware Contingency Planning 2502.06359]、[Risk Fields 2309.15501]、[Game-Theoretic Active Perception 2105.08169, Princeton]
- **室內 AMR**：**OA-MPC**（arXiv 2211.09156，Firoozi/Camps/**Schwager**, Stanford）——遮擋邊界 + 前向可達集合 + 終端停止約束保證遞迴可行性，**TurtleBot + LiDAR 真機**。用原始感測幾何，**不是學習式**。
  ★ **它的保守度旋鈕是假設的行人速度 `v_ped`，只測到 0.3/0.5/0.6 m/s，而真實行人 1.3 m/s**。全文 `freez` **0** 次、`conservat` 僅 2 次——**沒有討論調低假設的代價**。
  → **兩個陣營的旋鈕都沒有機率意義**：SCOPE 調熵門檻、OA-MPC 調假設速度。這是統合兩邊的論點。
- **交大 2024**：有**關鍵角（Critical Corner）**偵測 → 保持距離不貼牆轉彎，實測閃掉死角行人。但是**純幾何模組讀原始掃描，跟預測脫鉤**，且只抓得到幾何邊角（人擋人、玻璃、視野邊緣抓不到）。

## 未觀測區域的「預測」：有前案，但預測的是靜態結構
- **Occupancy Anticipation**（Ramakrishnan/Al-Halah/**Grauman**, ECCV 2020, arXiv 2008.09285）、**ProxMaP**（2305.05519）
- 但：預測**靜態結構**（牆、房間格局）、環境是**靜態掃描**（Gibson/MP3D，**沒有人在動**）、目的是**探索效率**不是安全、**無校準**
- ★ 這個區別解釋了方法為何要分兩塊：**靜態結構有建築規律可學；「櫃子後面有沒有人」沒有規律，學不出來，只能用可達性**

## 框架文件（報告時的主角之一）
**Principles and Guidelines for Evaluating Social Robot Navigation Algorithms**（arXiv 2306.16740, ACM THRI，2022 Social Navigation Symposium 數十位作者共識）
- 八原則，**P1 = Safety**
- 分類法明列「可見性是否受限於機器人感測器（含 **occlusions**、directionality、sensing delay）還是 ground truth」——**點名了維度，但沒操作化成指標**
- 該領域自承 benchmarking「can be biased to suit specific research narratives」
- ⚠ 以上是從 HTML 版抓的摘要，**引用前務必自己核對原文措辭**

其他入口：[awesome-robot-social-navigation](https://github.com/Shuijing725/awesome-robot-social-navigation)、Characterizing the Complexity of Social Robot Navigation Scenarios (2405.11410)

## 唯一乾淨的缺口
**佔據預測的機率從來沒有被驗證過涵蓋率。** 證據理論給的是 belief mass（表述），不是有限樣本涵蓋率保證。
SCOPE、台大 2022、Bosch 2024 三篇的 calibration/coverage/conformal 全部是 0。

相關：[[lidar-ogm-forecast-spec-v2]]、[[taiwan-thesis-prior-art]]、[[forecast-baseline-comparison]]
