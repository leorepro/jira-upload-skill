# jira-upload-skill

> 🌐 **Language:** 繁體中文 · [English](README.en.md)

一個 [Claude Code](https://claude.com/claude-code) skill，教 Claude 如何透過 REST API 把檔案上傳到 **Jira Cloud** 工單，並且安全處理 API token（不會把 `-u email:TOKEN` 暴露在命令列上）。

兩種使用情境都涵蓋：

- **使用 Claude Code 的人** — 安裝成 skill，跟 Claude 對話就能完成上傳。
- **不用 Claude Code 的人** — [`jira-upload-skill/SKILL.md`](jira-upload-skill/SKILL.md) 本身就是一份完整的 `curl` 操作指南，可以直接照著做。

---

## 它能做什麼

裝好之後，你可以對 Claude 說：

> 幫我把 `~/Downloads/report.pdf` 上傳到 `PROJ-123`

Claude 會：

1. 詢問缺漏的資訊（Jira site、登入 email、issue key、檔案路徑）
2. 檢查 `~/.netrc` 是否已經為你的 Jira site 設定好；如果沒有，會帶你完成設定
3. 執行 curl 上傳
4. 解析回傳的 JSON，確認附件確實掛上去

Skill 內建安全要求：你的 Atlassian API token 一律從 `~/.netrc`（chmod 600）或環境變數讀取，**絕不**直接寫在命令列。

---

## 安裝（Claude Code）

```bash
# Clone 或下載這個 repo
git clone https://github.com/leorepro/jira-upload-skill.git
cd jira-upload-skill

# 把 skill 目錄複製到 Claude Code 的 skills 路徑
mkdir -p ~/.claude/skills
cp -r jira-upload-skill ~/.claude/skills/
```

重啟 Claude Code（或開新 session）。Skill 會被自動偵測，之後只要請 Claude 把檔案上傳到 Jira，skill 就會自動觸發。

驗證安裝：

```bash
ls ~/.claude/skills/jira-upload-skill/SKILL.md
```

---

## 第一次使用前置作業（一次設定，終身受用）

需要準備 **兩樣**：

### 1. 取得 Atlassian API token

1. 前往 https://id.atlassian.com/manage-profile/security/api-tokens
2. 點 **Create API token**，給一個識別用的 label（例如 `claude-jira-upload`）
3. 產生後**立刻複製保存** — 視窗關閉就再也看不到

### 2. 設定 `~/.netrc`

```bash
cat >> ~/.netrc <<'EOF'
machine YOUR_SITE.atlassian.net
  login YOUR_EMAIL@example.com
  password YOUR_ATLASSIAN_API_TOKEN
EOF
chmod 600 ~/.netrc
```

設定完成後，所有對該 site 的 `curl --netrc` 都會自動帶上認證資訊。

---

## 不用 Claude Code 也能用

Skill 本身就是一份完整的 curl 操作手冊。最常用的一行：

```bash
curl -sS -X POST \
  -H "X-Atlassian-Token: no-check" \
  --netrc \
  -F "file=@/path/to/file.pdf" \
  "https://YOUR_SITE.atlassian.net/rest/api/3/issue/PROJ-123/attachments"
```

完整指南（多檔同時上傳、環境變數認證、錯誤排查、下載／刪除附件）請見 [`jira-upload-skill/SKILL.md`](jira-upload-skill/SKILL.md)。

---

## 適用範圍

- ✅ **Jira Cloud**（`*.atlassian.net`），使用 REST API v3
- ❌ Jira Server / Data Center — endpoint 與認證方式都不同，這個 skill 不適用
- ❌ Confluence 附件 — 是另一組 endpoint，不在本 skill 範圍

---

## 安全須知

- API token 等同密碼 — 不要 commit 進 git、不要貼到 chat、不要直接寫在命令列
- Token 一旦曝光，立刻到 [token 管理頁](https://id.atlassian.com/manage-profile/security/api-tokens) revoke 並重新產生
- 不同用途用不同 token（CLI 一個、CI 一個），出事時可以精準 revoke 受影響的範圍
- 建議每半年輪替一次

---

## 授權

MIT — 見 [LICENSE](LICENSE)。

## 貢獻

歡迎開 issue 或送 PR。如果你遇到 skill 沒有涵蓋的 Jira 邊角案例（自訂 CSRF 設定、附件大小限制、特殊區域 endpoint），請開 issue 回報。
