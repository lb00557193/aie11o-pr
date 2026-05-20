# Aiello PR Skills for Claude Code

三個 Claude Code slash commands，把開 Bitbucket PR 跟同步 Jira comment 的整套流程自動化：

| Command | 用途 | Source → Target |
|---------|------|-----------------|
| `/pr-dev <target>` | feature 上推 dev | 當前 branch → arg |
| `/pr-beta <source> <target>` | dev 推 beta | arg → arg |
| `/pr-master <source> <target>` | beta 推 master | arg → arg |

**Jira 連動：** 三個 command 都會在對應 Jira ticket 上維護**一條** marker comment。每次 cascade 跑完都會 PUT 覆寫，內容含當前 deploy stage（dev / beta / master），讓 PM / QA 從 Jira 一眼看出進度。Cascade 含多張 ticket 時，每張各自更新自己那條，不會混在一起。

---

## 前置需求

1. **Claude Code** 已裝
2. **`aie11o` CLI** 已裝（用來呼叫 Bitbucket API）
3. **兩張 Atlassian Classic API token**（一張 Bitbucket、一張 Jira）— 下方有建立步驟
4. 工作 repo 在 **Bitbucket Cloud**、**aiello_sharif** workspace 底下

---

## 安裝

### 1. Clone 進 Claude Code skills 目錄

```bash
git clone git@github.com:lb00557193/aie11o-pr.git /tmp/aie11o-pr
cp -R /tmp/aie11o-pr/pr-{dev,beta,master} ~/.claude/skills/
rm -rf /tmp/aie11o-pr
ls ~/.claude/skills/
# 應該看到 pr-dev/  pr-beta/  pr-master/
```

往後 skill 有更新時，重複跑上面的指令即可（會直接覆蓋舊版）。

### 2. 建立 Atlassian API tokens

⚠️ **重要：建 *classic* API token，不要建 *scoped* API token**

- ✅ Classic（無 scope 選擇步驟）→ 走 Basic auth，能通
- ❌ Scoped（有「Select app」+「Select scopes」步驟）→ 走 OAuth gateway，在我們 org 會被擋

#### 步驟

1. 開 <https://id.atlassian.com/manage-profile/security/api-tokens>
2. 點 **「Create API token」** 按鈕（**不是**「Create API token with scopes」）
3. 取名（例如 `claude-pr-bb` 和 `claude-pr-jira`），設 expiry（最長 366 天）
4. Copy token 字串（看起來像 `ATATT3xFfGF0...`）

建議建**兩張**：一張 Bitbucket、一張 Jira。也可以共用一張（unscoped classic 跨服務都通），但分開好管理、撤銷時較安全。

### 3. 設定 `~/.aie11o/aie11o.json`

如果你還沒設定過：

```bash
mkdir -p ~/.aie11o
```

新增或編輯 `~/.aie11o/aie11o.json`，加入這兩個 section（其他 section 如 `grafana` 不動）：

```json
{
  "bitbucket": {
    "workspace": "aiello_sharif",
    "email": "your.name@aiello.ai",
    "token": "<你的 Bitbucket classic API token>"
  },
  "jira": {
    "host": "aiello-eng.atlassian.net",
    "email": "your.name@aiello.ai",
    "token": "<你的 Jira classic API token>"
  }
}
```

### 4. 驗證

開 terminal 在任何 Bitbucket repo 底下跑：

```bash
aie11o bb whoami
```

應該印出你的帳號 email 和 workspace。沒過的話通常是 token 拼錯或不是 classic。

Jira 部分等下次跑 `/pr-dev` 時會自動驗，失敗會印 HTTP 狀態給你。

---

## 使用

### `/pr-dev <target>` — 從 feature branch 開 PR 到整合分支

```
/pr-dev dev
```

- **Source** = 當前 git branch（必須以 `feature/`、`feat/`、`fix/`、`bugfix/`、`hotfix/`、`improvement/`、`improvements/`、`chore/`、`refactor/`、`docs/`、`test/`、`release/`、`ci/` 之一開頭，否則跳出來請你明示）
- **Target** = 第一個 arg（你 repo 用什麼名都可以：`dev`、`develop`、`main`…）

流程：
1. 檢查 branch 是否已 push、是否已有重複 open PR
2. **抓 Jira ticket 的 summary / description / issuetype / priority** 當 context（失敗 fallback、不阻擋整個流程）
3. 從最新 commit subject + git diff + Jira context 自動生成 PR title 和 description（中文，照 `pr_generation` 模板：Why / How / Implement / 測試驗證 / 補充說明 / 相關連結）。**Why** 段優先用 Jira 描述、**How / Implement** 靠 diff
4. 建 PR
5. 把 description 同步成 Jira comment（含一個隱形 marker `<!-- claude-pr:AOX-XXXX -->`）。重跑同 ticket 時會用 prior PR description + 整段 feature 完整 diff 重新生成 holistic 描述，PUT 覆寫單一條 comment（不會多生）。

可選 flag：

```
/pr-dev dev --lang en      # 改用英文 template
```

### `/pr-beta <source> <target>` — Cascade

```
/pr-beta dev beta
```

兩個 arg 都要：source 跟 target。

- **掃 `git log origin/<target>..origin/<source>` 抓區間內全部 ticket**（不只最新一張），title 變 `[T1][T2]...[Tn] cascade <source> → <target>`、description 每張 ticket 各自一段
- 每張 ticket 的描述優先沿用其 source PR 的內容；source PR 描述為空時，自動從該 ticket 的 commit diff 重生
- **每張 ticket 的 Jira marker comment 都會 PUT 覆寫一次**，把 `**Stage:**` 行更新到 `beta`，並把這次 cascade PR 連結加進 `相關連結`

### `/pr-master <source> <target>` — 推到 production

```
/pr-master beta master
```

同 `/pr-beta` 邏輯，方向不同。Jira 各 ticket 的 marker comment 會把 Stage 推進到 `master`。

---

## 行為細節

### Jira comment 結構

每張 ticket 永遠只有**一條** marker comment（透過 `<!-- claude-pr:AOX-XXXX -->` 識別）。每次 PR 動作會 PUT 覆寫。內容固定佈局：

```markdown
**Stage:** <target>            ← 隨 cascade 進度更新（dev → beta → master）

## Why
...

## How
...

## Implement
- [x] ...

### 測試驗證
- [ ] ...

## 相關連結
- Jira: AOX-XXXX
- Dev PR: https://bitbucket.org/aiello_sharif/<repo>/pull-requests/<id>
- Beta PR: https://...
- Master PR: https://...
- Co-shipped tickets (cascade only): AOX-YYYY, AOX-ZZZZ
```

PM / QA 看 Jira 就知道：
- 這張票目前推到哪個環境（`Stage:`）
- 每個 stage 的 PR 連結
- Cascade 一起上的其他 ticket

ADF 渲染支援 headings、bullet / task lists、inline code、code blocks、links — 不會掉格式。

### 邊界情況

- ✅ 未 commit 的 `.DS_Store`、IDE 設定 — 忽略，不阻擋
- ⛔ Branch 沒 push 或落後 origin — 停下來告訴你 push 完再跑
- ⛔ 同 source → target 已有 open PR — 停下來告訴你 PR id + URL
- ⛔ Workspace 跟 `aie11o.json` 對不上 — 停下來告訴你（防止在錯誤 workspace 上跑）
- ⛔ Jira `jira` section 沒設 — Bitbucket PR 照建，Jira 步驟靜默跳過（不會 fail 整個流程）

### 安全限制（不會自動做的事）

- 不會 `git push`
- 不會 `aie11o bb pr merge`
- 不會自動 checkout
- 不會加 reviewers
- 不會 rollback PR（Jira 失敗時 PR 已建好，會把錯誤印給你決定）

---

## FAQ

**Q: 不是 Bitbucket 的 repo 能用嗎？**
A: 不行。只支援 Bitbucket Cloud。

**Q: 換到別的 Atlassian org 能用嗎？**
A: 可以。改 `~/.aie11o/aie11o.json` 的 `jira.host` 跟兩張 token 對應的帳號就行。WORKFLOW.md 不需要動。

**Q: 我的 branch 命名不在預設清單怎麼辦？**
A: 編 `~/.claude/skills/pr-dev/WORKFLOW.md` 步驟 2 的 regex，把你的前綴加進去。

**Q: PR / Jira 失敗會 rollback 嗎？**
A: 不會。Jira 失敗時 PR 已建好，會把 Jira 的錯誤訊息印出來給你看，你決定怎麼處理。

**Q: Cascade 含多張 ticket 時，Jira 留言會混在一起嗎？**
A: 不會。每張 ticket 各自有自己的 marker comment，cascade 會 loop per ticket、每張獨立 PUT。

**Q: 同一張 ticket 反覆發 PR 會把 Jira 灌爆嗎？**
A: 不會。marker 機制讓同張 ticket 永遠只有一條 comment，每次覆寫不新增。

**Q: 每次都要 approve permission 很煩**
A: Claude Code 設定 → 把常用的 Bash pattern 加進 `~/.claude/settings.json` 或專案 `.claude/settings.json` 的 `permissions.allow`。read-only 指令多半已經自動允許了。

---

## 結構

```
~/.claude/skills/
├── pr-dev/
│   ├── SKILL.md       # /pr-dev 入口
│   └── WORKFLOW.md    # 三個 command 共用的流程（含 ADF 轉換、step 8 Jira 邏輯）
├── pr-beta/
│   └── SKILL.md       # /pr-beta 入口，引用 pr-dev/WORKFLOW.md
└── pr-master/
    └── SKILL.md       # /pr-master 入口，引用 pr-dev/WORKFLOW.md
```

要客製改 `WORKFLOW.md` 即可，三個 skill 都會吃到改動。
