---
name: forecast-baseline-comparison
description: 訓練 CCLSTM 之前必做的預測 baseline 對照——persistence 與等速外插；它決定 T_f 該多長、網路值不值得。
metadata:
  type: project
---

2026-08-27 決定：**階段 4（錄資料）完成、階段 5（訓練）開始之前，先跑兩個 baseline 對照**，不要直接投入調網路。

**兩個 baseline：**
1. **persistence**——假設當前 occupancy 原地不動
2. **constant velocity**——用 flow 場做線性外插（這就是 Frozone 式 12 在做的事）

拿 CCLSTM 的預測跟這兩個比，在**不同 horizon** 上各比一次（0.5s / 1s / 1.5s / 2s）。

**Horizon 的經驗錨點（2026-08-27 從 TASE 論文全文讀到，比原本的猜測準）**：室內行人的最大互動範圍約 **t₀ ≈ 1.4 s**（室外約 2.4 s）——TASE 那篇就是照這個把 T_clip 設成 t₀=1.5 s。所以室內場景的關鍵 horizon 是 **1.4–1.5 s**，不是我原本說的 2–3 s。`T_f=10`（1 s）略短於這個門檻，延到 1.5–2 s 就夠，3 s 對室內是過度設計。

**Why:** 如果 CCLSTM 在 1 秒 horizon 上贏不過等速外插，那整張網路就是白費算力，Frozone 那種線性外插從一開始就是對的答案。量級估算支持這個疑慮：行人 1.3 m/s，1 秒走 1.3 m = 4cm 格上的 **33 格**（128 格網的 1/4，所以任務不 trivial）；但 **1 秒內人幾乎不轉彎也不停下**，等速外插在短 horizon 會強得驚人。學習模型真正拉開差距是在 2–3 秒以上——人要轉彎、停下、互相避讓，線性外插在那裡才會崩掉。

**How to apply:** 這個對照真正回答的問題不是「要不要做這個專案」，是「**網路從幾秒開始才值回票價**」。那個答案直接決定 [[bev-system-spec-v1]] 裡的 `T_f=10`（1 秒）夠不夠，還是必須照原計畫延到 2 秒。成本幾乎為零——資料本來就要錄，兩個 baseline 加起來十幾行 numpy。容易在忙著調網路時被忘掉，所以先記著。

相關：[[bev-system-spec-v1]]、[[cclstm-paper-spec]]、[[frozone-paper]]
