# Shared PR workflow

Referenced by the `pr-dev`, `pr-beta`, `pr-master` skills. Steps 1–6 are
shared; step 5 branches by which command invoked it.

---

## 1. Parse skill arguments

`$ARGUMENTS` may include any of these flags (anywhere in the string):

- `--lang en` / `--lang zh` — choose description language. Default is
  `zh` (Traditional Chinese).

Strip the flags out of `$ARGUMENTS` first; remaining tokens are the
positional source/target args.

## 2. Resolve source and target branches

Argument shape differs per command:

- **`/pr-dev <target>`** — one positional arg (target). Source = current
  HEAD, but only if HEAD matches a feature-style branch name. Accepted
  prefixes:
  `^(feature|feat|fix|bugfix|hotfix|improvement|improvements|chore|refactor|docs|test|release|ci)/`.
  Otherwise STOP and tell the user: "Current branch isn't a feature
  branch. Please specify both source and target explicitly."
- **`/pr-beta <source> <target>`** — two positional args. Both are
  required.
- **`/pr-master <source> <target>`** — two positional args. Both are
  required.

For any missing arg, ask the user once (e.g. "source branch?",
"target branch?") and wait for the reply. For `/pr-beta` and
`/pr-master`, the user may supply a single arg — interpret that as
target only if source is unambiguous from context; otherwise still ask
for the missing one.

Echo the resolved `<source> → <target>` back to the user before
proceeding (one line is enough — don't ask for confirmation, slash
command counts as authorization).

## 3. Resolve Bitbucket repo

```bash
git remote get-url origin
```

Parse the result (e.g. `git@bitbucket.org:aiello_sharif/tools_ui.git`)
into `<workspace>/<repo>`. The `aie11o` CLI uses repo slug only — the
workspace is implicit in `~/.aie11o/aie11o.json`.

If the remote isn't `bitbucket.org`, STOP and tell the user this skill
only supports Bitbucket Cloud.

If the parsed workspace doesn't match
`~/.aie11o/aie11o.json`'s `bitbucket.workspace`, STOP and tell the user
the aie11o config is configured for a different workspace.

## 4. Extract Jira ticket and fetch context

### 4a. Extract the ticket key

Regex `[A-Z]+-\d+` (case-sensitive) against:

1. The source branch name (e.g. `fix/AOX-5268-update-...` → `AOX-5268`).
2. If no match, `git log -1 --pretty=%s <source>`.
3. If still no match, ask the user.

Jira domain: `aiello-eng.atlassian.net` (NOT `aiello.atlassian.net`).

### 4b. Fetch ticket details from Jira (best-effort)

If `~/.aie11o/aie11o.json` has a `jira` section, fetch the ticket's
summary + description to use as context when generating the PR
description in step 6. **Failure here is non-fatal** — just skip and
proceed with diff-only generation.

```bash
JIRA_EMAIL=$(python3 -c "import json; print(json.load(open('$HOME/.aie11o/aie11o.json'))['jira']['email'])")
JIRA_TOKEN=$(python3 -c "import json; print(json.load(open('$HOME/.aie11o/aie11o.json'))['jira']['token'])")
JIRA_HOST=$(python3 -c "import json; print(json.load(open('$HOME/.aie11o/aie11o.json'))['jira']['host'])")

STATUS=$(curl -s -o /tmp/jira_issue.json -w "%{http_code}" \
  -u "$JIRA_EMAIL:$JIRA_TOKEN" \
  -H "Accept: application/json" \
  "https://$JIRA_HOST/rest/api/3/issue/<TICKET>?fields=summary,description,issuetype,priority,labels,components")
```

Handling:
- **200** — parse `summary`, `description` (ADF → plain text by walking
  `content[].content[].text`), `issuetype.name`, `priority.name`,
  `labels[]`, `components[].name`. Pass these as context to step 6.
- **401 / 403** — log "Jira fetch unauthorized — skipping context",
  continue without it.
- **404** — log "Jira ticket <TICKET> not found — skipping context",
  continue.
- **No `jira` section in config** — silently skip.
- Any other status — log it, continue.

The PR creation must NOT depend on this fetch succeeding.

### 4c. Plain text in the PR description

In the PR description body, write the ticket as **plain text** (e.g.
`AOX-5268`) — Bitbucket's Jira integration auto-linkifies it. Do NOT use
markdown links (`[X](url)`) or HTML anchors (`<a target="_blank">`):

- Markdown links render but open in the same tab.
- HTML anchors are not parsed — they display as literal text.
- Bitbucket has no way to force `target="_blank"` from PR description
  markdown. New-tab is a user-side action (⌘+click / middle-click).

In the PR description, write the ticket as **plain text** (e.g.
`AOX-5268`) — Bitbucket's Jira integration auto-linkifies it. Do NOT use
markdown links (`[X](url)`) or HTML anchors (`<a target="_blank">`):

- Markdown links render but open in the same tab.
- HTML anchors are not parsed — they display as literal text.
- Bitbucket has no way to force `target="_blank"` from PR description
  markdown. New-tab is a user-side action (⌘+click / middle-click).

## 5. Edge-case checks

Run in order; the first failure stops the workflow (except (a)).

**(a) Uncommitted local changes** — IGNORE. `.DS_Store`, IDE settings,
etc. are common and not blocking.

**(b) Branch not pushed / local ahead of remote** — check with:

```bash
git fetch origin <source>
LOCAL=$(git rev-parse <source>)
REMOTE=$(git rev-parse origin/<source> 2>/dev/null || echo MISSING)
```

If `REMOTE=MISSING` or `LOCAL != REMOTE` with local ahead, STOP. Tell
the user: "Branch `<source>` isn't pushed (or is ahead of origin). Push
first?" — do NOT auto-push.

**(c) Existing open PR for the same source → target** — query:

```bash
aie11o bb pr list <repo> --state OPEN -o json \
  | jq -r --arg s "<source>" --arg t "<target>" \
    '.values[] | select(.source.branch.name==$s and .destination.branch.name==$t)
     | "\(.id)\t\(.title)\t\(.links.html.href)"'
```

If any row returned, STOP. Show the user the existing PR id + URL.

## 6. Generate title and description

Two modes, based on which command:

### Mode A — `/pr-dev` (feature → integration)

**Title:**

```bash
SUBJECT=$(git log -1 --pretty=%s <source>)
```

If `SUBJECT` already starts with `[<TICKET>]`, keep verbatim. Otherwise
title = `[<TICKET>] <SUBJECT>`.

**Description** — generate based on the diff **and the Jira ticket
context fetched in step 4b** (if available). Follow the `pr_generation`
template structure. **Omit any section that has no real content**
(don't include empty headers or `<!-- ... -->` placeholders).

Inputs (in priority order):
1. **Jira ticket summary + description** (from step 4b) — primary source
   for the **Why** section. The PM-authored context typically explains
   the business reason / requirement better than what code alone shows.
2. **Git diff + commit subjects** — primary source for the **How** /
   **Implement** sections. The code shows what was actually changed.
3. **Jira issuetype / priority / labels** — light signal for tone
   (e.g. a Bug ticket → frame Why as "the bug was X").

If step 4b failed (no Jira context), generate from diff + commits only.

Diff context:

```bash
git fetch origin <target>
git log origin/<target>..<source> --pretty=format:"- %h %s"
git diff --stat origin/<target>...<source>
git diff origin/<target>...<source>   # full diff; skip if > 500 lines
```

Pick the template by the `--lang` flag from step 1 (default `zh`):

#### Chinese template (`--lang zh`, default)

```markdown
## Why
<根據 diff/commit/單號 簡述背景，2-3 句>

## How
<實作方法，1-2 句>

## Implement
- [ ] <主要變更 1>
- [ ] <主要變更 2>

### 截圖或錄影參考
<只在 diff 明顯涉及 UI 才寫；無則整段刪掉>

### 測試驗證
- [ ] <測試案例 1>
- [ ] <測試案例 2>

## 補充說明
<只在有限制 / TODO / 已知問題才寫；無則整段刪掉>

## 相關連結
- Jira: <TICKET>
```

#### English template (`--lang en`)

```markdown
## Why
<short background based on diff/commit/ticket, 2-3 sentences>

## How
<approach, 1-2 sentences>

## Implement
- [ ] <main change 1>
- [ ] <main change 2>

### Screenshots / Recordings
<only if diff clearly involves UI; otherwise omit this section>

### Test plan
- [ ] <test case 1>
- [ ] <test case 2>

## Notes
<only if there are limits / TODOs / known issues; otherwise omit>

## Links
- Jira: <TICKET>
```

Inline section guidance (applies to both languages):
- **Why** — short context; lean on commit message and Jira ticket name.
- **How** — what approach was taken (e.g. "restructure payload to nested
  object", "extract helper", "add migration"). Skip if trivial.
- **Implement** — bullet what files/features changed, not line-by-line.
- **Test plan / 測試驗證** — propose test steps the reviewer should run;
  tie to the user-facing flow when possible.

### Mode B — `/pr-beta` and `/pr-master` (cascade promotion)

A cascade PR may bundle multiple tickets. Build title + description by
**enumerating every ticket in the cascade** (same enumeration as step 8
cascade — `git log origin/<target>..origin/<source>` then regex
`[A-Z]+-[0-9]+`, sort -u), then for each ticket find its most recently
MERGED PR whose destination is the current source branch.

**Single-ticket cascade** — copy that PR's title and description
verbatim (legacy behavior).

**Multi-ticket cascade** — combine:

- **Title** — `[TICKET-A][TICKET-B]... cascade <source> → <target>`,
  ticket order matches the enumeration (sort -u alphabetical).
- **Description** — a header line stating "本 cascade PR 包含 N 個
  ticket: <list>（描述沿用各自最近合併進 <source> 的 PR：#A、#B）",
  then for each ticket a `---` separator and a `# [TICKET] <title>`
  section followed by that PR's description verbatim. Order matches
  the title.

Per-ticket lookup (with branch-name fallback for PRs whose title
doesn't have the `[TICKET]` prefix):

```bash
aie11o bb pr list <repo> --state MERGED -o json \
  | jq -r --arg s "<source>" --arg t "<TICKET>" \
    '[.values[]
      | select(.destination.branch.name==$s)
      | select((.title | test("\\[" + $t + "\\]")) or
               (.source.branch.name | test($t)))]
     | sort_by(.updated_on) | reverse'
```

Picking the best candidate (top wins):

1. Candidate with **non-empty `.description`** — use its title +
   description verbatim.
2. All candidates have empty descriptions → **fall back to diff-based
   generation** scoped to this ticket:
   - Find the ticket's commit range in the cascade scope:
     ```bash
     git log origin/<target>..origin/<source> --pretty=format:"%H %s" \
       | grep -E "\\[<TICKET>\\]|<TICKET>" \
       | tail -1   # earliest commit for the ticket
     # use the parent of that earliest commit as base, latest as tip
     ```
   - Run `git diff <base>...<tip>` to compute the ticket's contribution
   - Synthesize a Chinese description in the `pr_generation` template
     using only those commits + diff. Omit empty sections.
3. No candidate exists at all → log a one-line note
   (`"<TICKET> has no prior merged PR into <source>; skipped"`) and
   omit that ticket's section from the description (but **still
   include the ticket in the title** prefix list).

If NO ticket in the range had any source PR found, fall back to **Mode
A** (single diff-based description from the full cascade range) and
tell the user "no prior merged PRs found — generated from diff".

## 7. Create the PR

```bash
aie11o bb pr create <repo> \
  --title "<TITLE>" \
  --source <source> \
  --dest <target> \
  --description "<DESCRIPTION>"
```

Report the returned PR id and `links.html.href` URL on one line.

## 8. Post a Jira comment

All three commands run this step. The scope differs:

- **`/pr-dev`** — single-ticket scope. Use the ticket extracted in
  step 4.
- **`/pr-beta` / `/pr-master`** — cascade promotion. The PR may bundle
  multiple tickets, so extract ALL of them and **loop step 8 once per
  ticket** (each ticket gets its own holistic comment — never reuse
  one description across unrelated tickets). To enumerate:

  ```bash
  git fetch origin <source> <target>
  git log origin/<target>..origin/<source> --pretty=format:"%s" \
    | grep -oE "[A-Z]+-[0-9]+" | sort -u
  ```

  For cascade commands, substitute `origin/<source>` wherever the
  per-ticket logic below references `HEAD` (the holistic diff still
  spans `ORIGINAL_BASE..origin/<source>`).

The PR description on Bitbucket always describes **this PR's delta**
(the diff vs target). The Jira comment is different: it's a
**holistic** per-ticket description that captures the feature as a
whole, using prior PR descriptions + the full feature diff. On every
iteration (re-`/pr-dev`, cascade to beta, cascade to master) we
regenerate and overwrite the marker comment so a single Jira comment
always reflects the latest feature state plus current promotion stage.

### Required structure of the Jira comment

Every comment posted/overwritten in step 8 must contain — in this
order, at the **top** of the body:

```markdown
**Stage:** <target>
```

`<target>` is the destination branch of the PR being created in this
run:

| Command       | Stage value |
|---------------|-------------|
| `/pr-dev`     | the target arg (typically `dev` / `develop` / `main`) |
| `/pr-beta`    | the target arg (typically `beta` / `staging`) |
| `/pr-master`  | the target arg (typically `master` / `production`) |

This line is what makes the comment work as a deployment tracker: PM /
QA can read any ticket's Jira and instantly know how far the feature
has shipped.

Below the `Stage` line comes the holistic Why / How / Implement /
測試驗證 sections (regenerated per the re-iteration logic below). At
the bottom of the body, the **相關連結** section must list every PR
made for this ticket so far (dev / beta / master) so the comment
doubles as a deployment-status index. For cascade promotions
specifically, also list **co-shipped tickets** (other tickets bundled
in the same cascade PR) under 相關連結 — this lets QA see at a glance
what else lands together when they read any one ticket's Jira.

### Preconditions

- `~/.aie11o/aie11o.json` must contain a `jira` section:
  ```json
  "jira": { "host": "aiello-eng.atlassian.net", "email": "...", "token": "..." }
  ```
- The token must have `read:comment:jira`, `write:comment:jira` scopes.

If the `jira` section is missing, skip step 8 with a one-line note to
the user: "skipped Jira comment — no `jira` config". Do not fail the
overall flow.

### Detect whether this is the first or a re-iteration

Find prior PRs for the same ticket (any state — OPEN/MERGED/DECLINED),
excluding the PR just created in step 7:

```bash
aie11o bb pr list <repo> --state MERGED -o json > /tmp/_merged.json
aie11o bb pr list <repo> --state OPEN -o json > /tmp/_open.json
aie11o bb pr list <repo> --state DECLINED -o json > /tmp/_declined.json
jq -s --arg t "<TICKET>" --arg cur "<current-pr-id>" \
  '[.[].values[]] | map(select(.title | test("\\[" + $t + "\\]"))) | map(select(.id != ($cur|tonumber)))
   | sort_by(.created_on) | .' /tmp/_*.json
```

- If result is empty → **first-time path**.
- If non-empty → **re-iteration path**.

### First-time path

1. Use the **same description text** as the Bitbucket PR (step 6
   Mode A output).
2. Append the marker on its own line at the end:
   ```
   <!-- claude-pr:<TICKET> -->
   ```
3. Convert markdown → ADF (paragraph-per-line, v1 conversion below).
4. `POST` to Jira (see "API calls" below). Response `201` = success.

### Re-iteration path

1. **Fetch prior PR description** — take the earliest PR from the prior
   list and read its `.description` field. (We use the earliest because
   it usually has the most context; alternative: read all and merge.)
2. **Find the feature's original base commit**:
   ```bash
   FIRST_COMMIT=$(git log --all --grep "\[<TICKET>\]" --reverse --format=%H | head -1)
   ORIGINAL_BASE=$(git rev-parse "${FIRST_COMMIT}^")
   ```
3. **Compute the full feature diff** — from the original base up to
   current HEAD (covers everything done on this ticket across all
   iterations):
   ```bash
   git diff "$ORIGINAL_BASE"...HEAD
   git log "$ORIGINAL_BASE"..HEAD --pretty=format:"- %h %s"
   ```
4. **Regenerate a holistic description in Chinese** using three inputs:
   - prior PR description (carry forward the original Why / How
     framing where still accurate)
   - full feature diff (ensure Implement covers everything, not just
     the latest delta)
   - new commit subjects in this iteration (signal what the latest
     round changed)

   Output follows the same `pr_generation` template (Why / How /
   Implement / 測試驗證 / 相關連結), omit empty sections. This is a
   **separate** generated description from the Bitbucket PR's delta
   description — do not reuse the step-5 output.

5. **Locate the existing Jira comment** for this ticket (via marker):
   ```bash
   curl -s -u "$JIRA_EMAIL:$JIRA_TOKEN" \
     "https://$JIRA_HOST/rest/api/3/issue/<TICKET>/comment?maxResults=100" \
     | jq -r --arg ticket "<TICKET>" \
       '.comments[] | select(.body | tostring | contains("claude-pr:" + $ticket)) | .id' \
     | head -1
   ```
6. Append the marker `<!-- claude-pr:<TICKET> -->` to the regenerated
   description, convert to ADF.
7. **Write back:**
   - If a comment id was found → `PUT /rest/api/3/issue/<TICKET>/comment/<id>` (overwrite). Response `200` = success.
   - If no marker comment found (marker drift, manual edit, etc.) → `POST` a new one as in the first-time path. Tell the user "no prior marker comment found; posted a new one instead".

### Markdown → ADF conversion

Produce structured ADF (not plain text). Supports the markdown subset
the `pr_generation` template uses:

- `## heading`, `### heading` → ADF `heading` node (level from `#` count)
- `- [ ]` / `- [x]` → ADF `taskList` / `taskItem` (state TODO/DONE)
- `- item` / `* item` → ADF `bulletList` / `listItem`
- ` ```lang ... ``` ` → ADF `codeBlock` (language attr if present)
- `---` / `***` / `___` → ADF `rule`
- `**bold**` → text node with `strong` mark
- `` `inline code` `` → text node with `code` mark
- `[label](url)` → text node with `link` mark
- consecutive plain lines → `paragraph`

Reference Python:

```python
import re, uuid, json

def md_to_adf(md: str) -> dict:
    lines = md.split("\n")
    content = []
    i = 0
    while i < len(lines):
        line = lines[i]

        # Fenced code block
        if line.startswith("```"):
            lang = line[3:].strip()
            buf = []
            i += 1
            while i < len(lines) and not lines[i].startswith("```"):
                buf.append(lines[i])
                i += 1
            block = {
                "type": "codeBlock",
                "attrs": {"language": lang} if lang else {},
            }
            if buf:
                block["content"] = [{"type": "text", "text": "\n".join(buf)}]
            content.append(block)
            i += 1
            continue

        # Heading
        m = re.match(r"^(#{1,6})\s+(.*)$", line)
        if m:
            content.append({
                "type": "heading",
                "attrs": {"level": len(m.group(1))},
                "content": parse_inline(m.group(2)),
            })
            i += 1
            continue

        # Horizontal rule
        if line.strip() in ("---", "***", "___"):
            content.append({"type": "rule"})
            i += 1
            continue

        # Task list (consecutive lines)
        if re.match(r"^\s*-\s*\[[ xX]\]\s+", line):
            items = []
            while i < len(lines) and re.match(r"^\s*-\s*\[[ xX]\]\s+", lines[i]):
                mm = re.match(r"^\s*-\s*\[([ xX])\]\s+(.*)$", lines[i])
                items.append({
                    "type": "taskItem",
                    "attrs": {"localId": str(uuid.uuid4()),
                              "state": "DONE" if mm.group(1).lower() == "x" else "TODO"},
                    "content": [{"type": "paragraph", "content": parse_inline(mm.group(2))}],
                })
                i += 1
            content.append({"type": "taskList",
                            "attrs": {"localId": str(uuid.uuid4())},
                            "content": items})
            continue

        # Bullet list
        if re.match(r"^\s*[-*]\s+", line):
            items = []
            while i < len(lines) and re.match(r"^\s*[-*]\s+", lines[i]) \
                    and not re.match(r"^\s*-\s*\[[ xX]\]\s+", lines[i]):
                mm = re.match(r"^\s*[-*]\s+(.*)$", lines[i])
                items.append({
                    "type": "listItem",
                    "content": [{"type": "paragraph", "content": parse_inline(mm.group(1))}],
                })
                i += 1
            content.append({"type": "bulletList", "content": items})
            continue

        # Empty line → skip (paragraphs separate naturally)
        if line.strip() == "":
            i += 1
            continue

        # Paragraph
        buf = [line]
        i += 1
        while i < len(lines) and lines[i].strip() != "" \
                and not lines[i].startswith("#") \
                and not lines[i].startswith("```") \
                and not re.match(r"^\s*[-*]\s+", lines[i]) \
                and lines[i].strip() not in ("---", "***", "___"):
            buf.append(lines[i])
            i += 1
        content.append({"type": "paragraph",
                        "content": parse_inline(" ".join(buf))})

    return {"type": "doc", "version": 1, "content": content}


def parse_inline(text: str) -> list:
    pattern = re.compile(r"(`[^`]+`)|(\*\*[^*]+\*\*)|(\[[^\]]+\]\([^)]+\))")
    nodes, pos = [], 0
    for m in pattern.finditer(text):
        if m.start() > pos:
            nodes.append({"type": "text", "text": text[pos:m.start()]})
        s = m.group(0)
        if s.startswith("`"):
            nodes.append({"type": "text", "text": s[1:-1], "marks": [{"type": "code"}]})
        elif s.startswith("**"):
            nodes.append({"type": "text", "text": s[2:-2], "marks": [{"type": "strong"}]})
        else:  # link
            inner = re.match(r"\[([^\]]+)\]\(([^)]+)\)", s)
            nodes.append({
                "type": "text",
                "text": inner.group(1),
                "marks": [{"type": "link", "attrs": {"href": inner.group(2)}}],
            })
        pos = m.end()
    if pos < len(text):
        nodes.append({"type": "text", "text": text[pos:]})
    return nodes if nodes else [{"type": "text", "text": text or " "}]
```

The marker `<!-- claude-pr:<TICKET> -->` is plain text — it'll render as
a small visible string at the end of the comment. That's acceptable
(humans can tell it's a system marker; the regex still finds it for
overwrite).

### API calls

```bash
JIRA_EMAIL=$(python3 -c "import json; print(json.load(open('$HOME/.aie11o/aie11o.json'))['jira']['email'])")
JIRA_TOKEN=$(python3 -c "import json; print(json.load(open('$HOME/.aie11o/aie11o.json'))['jira']['token'])")
JIRA_HOST=$(python3 -c "import json; print(json.load(open('$HOME/.aie11o/aie11o.json'))['jira']['host'])")

# POST (first time)
curl -s -o /tmp/jira_comment.json -w "%{http_code}\n" \
  -u "$JIRA_EMAIL:$JIRA_TOKEN" \
  -X POST -H "Content-Type: application/json" \
  -d "$BODY" \
  "https://$JIRA_HOST/rest/api/3/issue/<TICKET>/comment"

# PUT (re-iteration overwrite)
curl -s -o /tmp/jira_comment.json -w "%{http_code}\n" \
  -u "$JIRA_EMAIL:$JIRA_TOKEN" \
  -X PUT -H "Content-Type: application/json" \
  -d "$BODY" \
  "https://$JIRA_HOST/rest/api/3/issue/<TICKET>/comment/<comment-id>"
```

### Response handling

- `200` / `201` — success. Report: "✅ Jira comment posted/updated on <TICKET>".
- `403` — scope/permission issue. Surface the Jira error verbatim.
- `404` (POST) — ticket not found; verify ticket key from step 3.
- `404` (PUT) — comment was deleted between the search and the write;
  fall through to POST a new one.
- Other — surface the response body, do NOT fail the overall flow
  (PR is already created).

Never let a Jira failure roll back or undo the Bitbucket PR. Worst case
the user gets a created PR plus a Jira error message they can act on.

---

## Notes

- All `aie11o bb` commands take the repo slug only — never pass the
  workspace.
- No reviewers added (user opted out).
- Don't auto-merge or auto-push under any condition.
- If `aie11o` isn't installed or auth fails, surface the error verbatim.
