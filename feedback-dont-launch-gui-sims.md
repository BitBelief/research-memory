---
name: feedback-dont-launch-gui-sims
description: 使用者要自己啟動模擬器／GUI 程式，不要替他開；改完檔案就給指令讓他自己跑
metadata: 
  node_type: memory
  type: feedback
  originSessionId: fd444371-fc80-4fb2-9886-b7574d5f1718
  modified: 2026-10-19T08:53:48.397Z
---

2026-10-19 使用者明確表態：「我要自己開這些模擬器你不要幫我開」。

**How to apply**：改完 world／launch／節點之後，**只給指令**，不要用 Bash 背景跑
`ros2 launch`、`gz sim` 或任何會開視窗的東西。驗證要靠讀檔、產生器重跑、SDF 解析
（`gz sdf -p`）、離線推演狀態機這類不開 GUI 的手段。如果真的非跑不可才能驗證，
先問過再跑。

**Why**：
1. 模擬器開在他的螢幕上、吃他的 GPU，他要自己掌握什麼時候開、開幾個。
2. 我從沙箱啟動吃不到 RTX 4060（EGL 失敗、退回軟體算繪），量到的 RTF 根本不準
   （我量 74.3%，對他沒有參考價值），反而製造假資訊。
3. 殘留程序會互相搶 GPU 並且重複發 /l2/points、/clock —— 這本來就是他
   [[l2-3d-dynamic-static-pivot]] 專案裡 RTF 亂跳的已知原因，我不該再製造更多。

純物理、不含 gpu_lidar 的 headless（`gz sim -s -r`）在沙箱可行，是少數例外，
但一樣先問。
