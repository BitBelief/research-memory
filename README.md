# 研究記憶庫

Claude Code 的持久記憶檔。每個 `.md` 是一則事實，`MEMORY.md` 是索引（每則一行）。

**索引先讀 [MEMORY.md](MEMORY.md)。現行技術方向與全部實驗結果在 [lidar-ogm-forecast-spec-v2.md](lidar-ogm-forecast-spec-v2.md)。**

## 慣例

- 被取代的檔案**不刪**，在索引行標 `⚠ 已被 X 取代`，保留當時的判斷理由與教訓。
- 日期一律寫絕對日期，不寫「上週」「最近」。
- 數字只寫在產生它的那個檔案裡，不在別處複製一份——複製的那份一定會先過期。
- 一則事實一個檔案。跨檔關聯用 `[[檔名]]` 連結。

## 在新機器上還原

```bash
git clone <repo> ~/research-memory
ln -s ~/research-memory ~/.claude/projects/-home-user/memory
```

Claude Code 不會自己同步這個資料夾，跨機器靠這個 repo。改完記得 commit。

## 為什麼要有版本控制

規格改過方向（雙目相機 → 2D 光達、筆電當大腦 → Orin Nano 全上車），數字也被自己的檢驗推翻過一次
（座標系錯位產生的假發現）。寫論文時會需要回答「這個數字是哪個版本、修正前還是修正後」，
git log 是唯一可靠的答案。
