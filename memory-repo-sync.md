---
name: memory-repo-sync
description: 記憶資料夾本身是 git repo，remote 在 GitHub private repo BitBelief/research-memory，改完記憶檔要 commit
metadata: 
  node_type: memory
  type: reference
  originSessionId: 7c07902a-42c5-4752-a81b-65d86ee9fe86
  modified: 2026-09-15T02:57:49.815Z
---

記憶資料夾 `~/.claude/projects/-home-user/memory/` **本身就是一個 git repo**（2026-09-15 建立）。

- remote：`https://github.com/BitBelief/research-memory`（**private**，帳號 `BitBelief`）
- 分支 `main`，認證走 gh CLI 的 HTTPS credential helper（已設好，push 不需輸入帳密）
- git 身分設在 repo 內（`user.name=jam4413256`），沒有全域 gitconfig

**How to apply**：新增或修改記憶檔之後，順手 commit 並 push，否則只存在本機。

```bash
git -C ~/.claude/projects/-home-user/memory add -A && git -C ~/.claude/projects/-home-user/memory commit -m "..." && git -C ~/.claude/projects/-home-user/memory push
```

**Why**：Claude Code 的記憶是純本機檔案，不會同步到帳號，換機器就沒了。而且研究方向被自己的
檢驗推翻過（座標系錯位造成的假發現，見 [[lidar-ogm-forecast-spec-v2]]），寫論文時需要回答
「這個數字是哪個版本」，git log 是唯一可靠來源。

commit message 用來記錄「這個 commit 是哪個階段」，數字本身留在各自的記憶檔，不要複製進
commit message 以外的地方。慣例寫在 repo 的 `README.md`。
