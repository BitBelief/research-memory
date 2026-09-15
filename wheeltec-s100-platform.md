---
name: wheeltec-s100-platform
description: 車體＝WHEELTEC S100 差速底盤；控制介面是 /cmd_vel，底層全現成；這條線正好是「Xavier 跑驅動、筆電當大腦」的切點。
metadata: 
  node_type: memory
  type: project
  originSessionId: 505a0658-2b00-4093-b0cd-bbfa5bbab5cd
  modified: 2026-09-15T04:20:24.797Z
---

2026-09-09 使用者確認：[[lidar-ogm-forecast-spec-v2]] 裡那台「差速自走車」＝ **WHEELTEC S100**（轮趣科技／東莞，差速服務機器人，支援自動回充）。

**控制鏈路（不需自己實作）**
```
/cmd_vel (Twist: v, ω) → wheeltec_robot 節點（差速運動學）
  → USB 序列，幀頭 0x7B / 幀尾 0x7D、XOR 校驗、24 byte 回傳幀
  → STM32（PID + 編碼器）→ 馬達
  → 回傳 /odom + /imu/data_raw
```
官方套件 `turn_on_wheeltec_robot`。**以機器上的原始碼為準**（各版本不一致）。

**★ 協定實體（2026-09-09 自 `1417265678/wheeltec` 鏡像實讀，阿克曼版；差速版幀格式相同、callback 收 Twist）**
送（11B）：`[0]=0x7B [1]產品型號 [2]使能 [3-4]Vx [5-6]Vy [7-8]ωz [9]XOR(0..8) [10]=0x7D`；速度皆為 **short 大端、原值×1000**（m/s→mm/s、rad/s→mrad/s）。
收（24B）：`[0]=0x7B [1]Flag_Stop [2-7]底盤XYZ速度 [8-13]加速度XYZ [14-19]陀螺XYZ [20-21]電壓 [22]XOR(0..21) [23]=0x7D`。
換算常數：`GYROSCOPE_RATIO=0.00026644`(=1/65.5/57.3, FS_SEL=1 ±500°/s)、`ACCEl_RATIO=16384.0`(±2g)。

**⚠⚠ 三個對論文有直接影響的發現**
1. **IMU＝MPU6050（6 軸，無磁力計）**，不是 N100。→ **yaw 只能陀螺儀積分、必然漂移**。錄製對策：每段控制在數分鐘內，或用光達 scan matching 校正航向。
2. **★ `/odom` 的 covariance 是硬編碼常數**（`odom_pose_covariance[36]` 全是 1e-3/1e6/1e-9 魔術數字），**不是真的不確定度估計**。本題目主軸就是「校準過的不確定度」——**絕不可拿它當位姿不確定度用**，要自己估或明講不用。
3. 有 `robot_pose_ekf.launch` → **不是純輪速積分，有融合 IMU**，但 `robot_pose_ekf` 是 ROS1 老套件（現代應用 `robot_localization`）。
⚠ 官方文件通篇 `ROS_MASTER_URI`/`roscore` → **WHEELTEC 的堆疊是 ROS 1**。若真要換 Orin/Jazzy，底盤驅動不是「重編」而是**重寫**。

**★ 對論文的意義：`/cmd_vel` 就是分工線**
線以上（筆電 Jazzy）＝OGM 預測／遮擋推理／校準／規劃；線以下（Xavier NX Foxy/Galactic）＝底盤驅動＋RPLIDAR A1 驅動。
→ **跨 ROS 版本互通只需通四個 topic**：`/scan`、`/odom`、`/imu/data_raw` 上行，`/cmd_vel` 下行。連通性測試先只測這四個，別測「整個系統相容」。

**⚠ 2026-09-14 分工線解消**：使用者決定改用 **Orin Nano，全部運算搬上車**（筆電放車上太危險；且 WiFi 當控制迴路在密集人流下不安全）。→ **沒有跨版本邊界了**，「先測四個 topic」這個待辦作廢；筆電只剩開發／訓練／離線分析，兩邊唯一介面是 **bag 檔**。細節與電源風險見 [[lidar-ogm-forecast-spec-v2]]。

**錄製紀律補充**：`/odom` 與 `/imu/data_raw` 必須跟 `/scan` 一起錄。動靜態判斷要扣 ego-motion，自身運動估不準 → 靜態牆被判成在動，「車在動」那批資料會整批報廢。

**★★ 直連筆電已實測成功（2026-09-09）——「跳過 NX，筆電直接控」這條路走得通**
接法：底盤 USB → 筆電，晶片是 **WCH CH343（`1a86:55d4`）→ 列舉成 `/dev/ttyACM0`**（CDC-ACM，**不是** ttyUSB）。使用者已在 `dialout` 群組，免 sudo。**115200 8N1**。
實測：5 秒 **100 有效幀 / 0 校驗失敗**、**回傳 20.0 Hz**、**加速度 Z=+1.00 g（重力，解碼正確的鐵證）**、陀螺≈0、**電池 25.40 V（6S/24V 系統）**。上面那份協定一字不差。
另有次要幀：**8 bytes，頭 `0x7C` 尾 `0x7F`**，XOR 同規則，目前內容全零，用途未知。
⚠ 排錯教訓：一開始 11 種鮑率全收 0 bytes，寫入只回 `ff ff` 之類毛刺 → 那是**線沒接對**，不是鮑率問題。接對後立刻正常。**全 0xFF/0xFE 雜訊 = 實體沒接通，別再掃鮑率。**
解碼腳本：`~/wheeltec_test/read_chassis.py`（`python3 read_chassis.py <秒數>`）。
★ **時間軸**：odom/IMU **20 Hz**、光達 **7.8 Hz**（且抖動 ±3.5%）→ **管線必須把 odom 內插到光達時間戳**，不可一對一配對。錄 bag 時各帶自己的 timestamp，對齊離線做。
**★★ 寫入與失控測試也通過（2026-09-09，輪子架高）——端到端驗證完成**
送 `vx=0.100` → 實測 `+0.087~+0.099 m/s`，約 0.2 s 收斂；送零 → 0.2 s 內停。
★ **STM32 有看門狗：斷訊約 1.0 秒後自動歸零。**
→ 驅動設計兩條鐵律：① **必須固定 10~20 Hz 重送指令**（送一次就不管會走走停停）② **1 秒太久**（0.5 m/s＝滑行 50 cm，密集人流裡不可接受）→ **自己的節點要更短超時，如 0.2 s 沒收到 `/cmd_vel` 就主動送零**。
⚠ **指令純直線（wz=0）時實測 wz 在 ±0.02 rad/s 跳**＝左右輪轉速不一致。落地後累積成**航向漂移**，與 MPU6050 無磁力計（yaw 只能陀螺積分）**兩個誤差疊加 → 「車在動」那批資料的最大風險**。
腳本：`~/wheeltec_test/motion_test.py`（`finally` 保證送零）。
**下一步順序（已議定）**：① **先做輪徑/輪距校正**（直線 2m＋原地轉 10 圈，需落地）→ ② 再包 ROS 2 節點（Jazzy、自己的 package、MIT），因為節點參數要填校正後的值 → ③ 接光達錄第一份 bag。

**★★ 2026-09-14 三個會重複發生的除錯教訓**
1. **串口是獨佔的，第一個動作永遠是 `fuser -v /dev/ttyACM0`。** 殘留的 python（尤其被 `Ctrl+Z` **暫停**而非 `Ctrl+C` 結束的）會一直握著埠。症狀：串口開得起來但讀不到資料；或 C++ 端丟 `serial::SerialException: device reports readiness to read but returned no data`。實測曾有 3 個殘留 process 把 byte 流分掉 → 自己的腳本（逐 byte 滑動分幀）只是幀率減半仍有輸出，官方 driver（盲讀固定 24B）則 100% 失敗。**不要去猜鮑率或線材。**
2. **底盤會卡在 latched stop。** 指令送得出去、24B 回傳幀正常、IMU／電壓全對（25.3V），但編碼器回報 `vx=+0.000`、馬達完全不轉。**解法＝底盤重新送電**（2026-09-14 實證有效）。判別點：回傳正常但 `vx` 恆 0 ＝ 邏輯路正常、馬達路沒通。
3. **`motion_test.py` 的看門狗判定在 `vx` 恆為 0 時是假陽性**（條件 `abs(vx)<0.01 and el>0.3` 必然在 0.4s 觸發，印出「0.4 秒看門狗」）。**真值是 2026-09-09 量到的 1.0 秒。**

**★★ 官方 ROS 2 driver 實測（2026-09-14）——`robotverseny/drivers` fork，已是 Jazzy**
位置 `~/wheeltec_ws/src/drivers/`（非 WHEELTEC 原廠，是匈牙利 robotverseny 的 fork）。→ **推翻舊筆記「WHEELTEC 驅動是 ROS1/Foxy 時代、不保證 Jazzy 編得過」**。
編譯：`colcon build --symlink-install --packages-skip udp_joystick_ros`（該包相依 `roscpp`/`message_generation` ＝ ROS 1，**必然編不過**，README 自己的清單也跳過它）。rosdep 需 `--skip-keys "libpcl-all libpcl-all-dev roscpp message_generation message_runtime serial"`。
- 送出幀格式**與自行解碼完全一致**；`tx[1]`/`tx[2]` 寫死 0 → **「產品型號／使能」對差速版不重要**（自己的 `motion_test.py` 也送 0 且能動，已交叉驗證）。
- ⚠ **`spin_some()` 寫在 `if(Get_Sensor_Data())` 內**（`wheeltec_robot.cpp:434`）→ **讀不到回傳幀就完全不處理 `/cmd_vel`，而且不報任何錯**。「沒有 `/imu`」與「輪子不轉」是同一個原因的兩個症狀。
- ⚠⚠ **實測 `/imu` 只有 6.67 Hz ＝ 20/3**（間隔穩定 0.15s）。推論：底盤每 50ms 送 24B 主幀＋**8B 次要幀（頭 `0x7C` 尾 `0x7F`）**＝32B ＝ 640 B/s；driver 盲讀固定 24B → 26.67 次/秒，只有 1/4 對齊成功 → 6.667 Hz。**尚未用 bytes/s 實測驗證。** 後果：`/cmd_vel` 只以 6.67Hz 被處理（發 >6.67Hz 會 queue 積壓成延遲）、odom 掉 2/3 樣本、**且 20Hz 是「odom 內插到光達時間戳」那條錄製規格的前提，現在比光達還慢**。
- ⚠ **topic 名稱與舊假設不同**：`odom_combined`（非 `/odom`）、`imu`（非 `/imu/data_raw`）→ **錄 bag 清單必須改，否則錄到空的**。
- ⚠ 裸 `ros2 run` 時 `serial_baud_rate` **預設 9600**（launch 才給 115200）；`akm_cmd_vel` 必須是 `none`，否則 `/cmd_vel` 被**靜默丟棄**。
- ⚠ 沒有速度上限、沒有固定頻率重送、**沒有 0.2s 安全超時** → 三條鐵律都要自己做。
→ **傾向用自己的節點**（`~/s100_ws/src/wheeltec_s100_driver`）：分幀邏輯本來就對，且超時／topic 命名／covariance 都是論文會被問的細節。

**★★ 已定案（2026-09-14 晚）：底盤控制走自己的 `wheeltec_s100_driver`，官方那份只留作參考。**
實測 `/imu/data_raw` **20.1 Hz**（官方 6.67 Hz）→ **分幀問題結案**。官方 `~/wheeltec_ws` 仍要留著，因為 **M10P 光達的 `lslidar_driver` 在那裡**。
⚠ `ros2 topic hz` 顯示 `min 0.001s / max 0.052s`：**確實有多幀擠在同一次 read 到達**（批次比預期常見）。發布的牆鐘時間本來就抖，重點是 header 時間戳要均勻——見下方修正。
**2026-09-14 對自有節點做的三處修正（已驗：語法、ROS import、6 個單元測試全過；批次行為僅真機可驗）**
1. **`rx_loop` 批次時間戳**：原本每幀蓋「解析當下」時鐘 → 批內 `dt≈0`，位移被積分吃掉。改成把「上一批→這一批」的實際間隔**平均分配給批內各幀**（總時間守恆），並改用整數奈秒避免 float 在 1.7e9 秒量級的精度損失。
2. **`dt ≥ 0.5s` 不再靜默跳過**：會 warn 並累計 `_gap_drops`。里程少掉一段位移必須看得見。
3. **check-then-use 競態**：`rx_loop` / `tx_tick` / **`shutdown`** 三處改成先取本地 `ser = self._ser` 再用，並把 `AttributeError` 加進 except。`shutdown` 那處最關鍵——**停車路徑拋例外的話零速度送不出去**。
**⬜ 仍未處理（已知，不急）**：① **從非 ROS 執行緒 publish**（`rx_loop`→`on_state`→`publish`/TF）——rclpy publisher 非 thread-safe，**若日後關閉時隨機當掉第一個查這裡**，正解是丟 queue 由 timer 發布 ② 次幀未驗 XOR ③ `angular_velocity_covariance`/`linear_acceleration_covariance` 全 0（餵 `robot_localization` 前要填）④ `max_linear` 的 clamp 在除以 `lin_scale` 之前。
**★ 版控（2026-09-15）**：`~/s100_ws` 已是 git repo，remote `https://github.com/BitBelief/s100_ws`（**private**，帳號 `BitBelief`）。只收 `src/`＋README＋.gitignore 共 13 檔；`build/ install/ log/ __pycache__/ .pytest_cache/` 全部忽略（colcon 可重生）。Orin Nano 上直接 `git clone` 即可，不用手動搬檔。記憶庫版控見 [[memory-repo-sync]]。

**⚠⚠ 錄任何 bag 之前必做**：`config/s100.yaml` 的 `linear_scale` / `angular_scale` 仍是 1.0 —— **輪徑/輪距校正（直線 2m ＋ 原地轉 10 圈，需落地）還沒做**。不做的話里程有系統性比例誤差，ego-motion 扣不乾淨。

**⚠ 桌上那張 Jetson 載板是 Waveshare JETSON-IO-BASE-A**（Nano/Xavier NX 世代）：**只吃 5V**（官方配 5V/4A，桶狀 5.5mm OD × 2.1mm ID 中心正極），**且無 J48 電源選擇跳線**，DC 座直通電路。曾誤插 19V/3.42A（Orin Nano 官方變壓器）→ 當下無反應，改用 USB 供電才開機；模組存活。5V/4A 對 Xavier NX 15W 模式是**剛好卡上限**，建議 5A。
