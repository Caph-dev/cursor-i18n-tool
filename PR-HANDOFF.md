# Handoff: 向 cursor-i18n-tool 提交 PR（Tab 键位误替换修复）

**Date:** 2026-05-26  
**仓库路径:** `~/Projects/cursor-i18n-tool`  
**上游:** https://github.com/Wuyf5275/cursor-i18n-tool  
**你的 GitHub:** `Caph-dev`（`gh` 已登录）  
**当前状态:** `main` 上有未提交改动；`origin` 指向上游（尚无 fork）

---

## 一句话任务

Fork 上游 → 新建分支 → **只提交 Tab/键位相关修复**（推荐）→ push 到你的 fork → `gh pr create` 指向上游 `main`。

---

## 本 PR 要解决的问题（写给维护者）

汉化后 `workbench.desktop.main.js` 键盘扫描表中的 `"Tab"` 被改成 `"Tab 补全"`：

```text
# 汉化前
[1,49,"Tab",2,"Tab",9,"VK_TAB"]

# 汉化后（错误）
[1,49,"Tab 补全",2,"Tab 补全",9,"VK_TAB"]
```

renderer 日志：

```text
[KeybindingService]: Keyboard event cannot be dispatched
Converted keydown event - keyCode: 2 ('Tab 补全')
```

**根因：** `riskyShortWords` 中 `"Tab" → "Tab 补全"` 的 `jsxRegex` 误匹配 `49,"Tab"`（数组元素，非 UI 文案）。

**修复要点：**

1. 从 `riskyShortWords` 移除 `"Tab"`；设置侧栏用 `scopedReplacements` 的 `tab:"Tab"` → `tab:"Tab 补全"`。
2. 新增 `isProtectedKeybindingContext()`，在 `VK_*`、扫描表 `[n,n,"Key",...]` 等上下文跳过危险短词替换。
3. 扩展 UI 属性白名单：`markdownDescription`、`aria-label` 等。

---

## ⚠️ 提交范围建议（重要）

当前工作区 `git diff` **不止 Tab 修复**，还包含大量 `safeGlobalDict` 新词条、`scopedReplacements` 整块、备份逻辑改动等。

| 策略 | 说明 |
|------|------|
| **推荐：聚焦 PR** | 只提交 Tab/键位修复，上游更容易合并 |
| 可选：大 PR | 词典扩充 + Tab 修复一起提，需在 PR 描述里分段说明 |

下面命令按 **聚焦 PR** 编写。若你要提交全部本地改动，见文末「附录：提交全部改动」。

---

## 步骤 0：在 Cursor 中打开正确目录

```text
~/Projects/cursor-i18n-tool
```

确认：

```bash
pwd   # → .../cursor-i18n-tool
git status
```

---

## 步骤 1：Fork 上游（仅需一次）

浏览器或 CLI：

```bash
gh repo fork Wuyf5275/cursor-i18n-tool --clone=false
```

会在 `Caph-dev/cursor-i18n-tool` 创建 fork。

添加 fork 为 push 远程（保留 `origin` = 上游）：

```bash
cd ~/Projects/cursor-i18n-tool
git remote add fork git@github.com:Caph-dev/cursor-i18n-tool.git
# 若已存在：git remote set-url fork git@github.com:Caph-dev/cursor-i18n-tool.git
git remote -v
```

期望：

```text
origin  → Wuyf5275/cursor-i18n-tool   (fetch/push 只读上游亦可)
fork    → Caph-dev/cursor-i18n-tool    (push PR 分支)
```

---

## 步骤 2：同步上游 main

```bash
git fetch origin
git checkout main
git pull origin main
```

---

## 步骤 3：创建 PR 分支

```bash
git checkout -b fix/tab-keybinding-scan-table
```

---

## 步骤 4：只暂存 Tab/键位相关文件（聚焦 PR）

**核心改动文件：**

- `src/dict.js` — 删除 `riskyShortWords` 中的 `"Tab"`，加注释
- `src/i18n-core.js` — `isProtectedKeybindingContext`、guard 包装、UI props 扩展

若 `dict.js` 里还混有大量**无关**新词条，有两种做法：

### 做法 A：整文件提交（简单）

若你愿意把 `dict.js` 里其它新词条也放进同一 PR，直接：

```bash
git add src/dict.js src/i18n-core.js
```

在 PR 描述里写清楚「含词典扩充」。

### 做法 B：只提交 Tab 相关 hunk（推荐给上游）

```bash
# 先只 add i18n-core（Tab 修复逻辑全在此 + dict 删 Tab）
git add src/i18n-core.js

# dict.js 用交互式暂存，只选：删除 "Tab" 行 + 注释块
git add -p src/dict.js
# 对每个 hunk：y=暂存 n=跳过
# 跳过 safeGlobalDict 大块新增（若不想放进本 PR）
```

**`i18n-core.js` 里与 Tab PR 强相关的部分：**

- `isProtectedKeybindingContext`
- `uiProps` 扩展
- 危险短词循环里的 `guard(...)`
- `scopedReplacements` 中的 `['tab:"Tab"', 'tab:"Tab 补全"']`（若上游尚无）

**可不放进 Tab PR 的部分（另开 PR 或后续提交）：**

- `safeGlobalDict` 大量新键
- `scopedReplacements` 其它设置项
- `backupFile` 行为变更（「保留当前文件继续汉化」）

---

## 步骤 5：提交

```bash
git commit -m "$(cat <<'EOF'
fix: prevent Tab keybinding breakage from risky short-word i18n

Remove "Tab" from riskyShortWords so jsxRegex no longer rewrites
keyboard scan table entries like [1,49,"Tab",2,"Tab",9,"VK_TAB"].

Add isProtectedKeybindingContext() to skip replacements near VK_*,
scan-code arrays, and KeyCode metadata. Settings sidebar label
remains translated via scoped tab:"Tab" → tab:"Tab 补全".

Fixes Cursor Tab / IntelliSense accept failing with
"Keyboard event cannot be dispatched" after localization.
EOF
)"
```

---

## 步骤 6：Push 到 fork

```bash
git push -u fork fix/tab-keybinding-scan-table
```

---

## 步骤 7：创建 PR

```bash
gh pr create \
  --repo Wuyf5275/cursor-i18n-tool \
  --base main \
  --head Caph-dev:fix/tab-keybinding-scan-table \
  --title "fix: prevent Tab keybinding breakage from risky short-word i18n" \
  --body "$(cat <<'EOF'
## Summary

After running the localization tool, **Tab** stopped accepting Cursor Tab / IntelliSense suggestions. Focus jumped to other UI controls instead.

**Root cause:** `riskyShortWords` mapped `"Tab"` → `"Tab 补全"`. The `jsxRegex` path incorrectly matched keyboard scan table tuples such as `49,"Tab"` inside:

```text
[1,49,"Tab",2,"Tab",9,"VK_TAB"]
```

This broke `KeybindingService` dispatch (`Keyboard event cannot be dispatched`, keyCode `Tab 补全`).

## Changes

- Remove `"Tab"` from `riskyShortWords` (settings nav still uses scoped `tab:"Tab"` → `tab:"Tab 补全"`).
- Add `isProtectedKeybindingContext()` to skip risky replacements near `VK_*`, scan-code arrays, and keybinding metadata.
- Extend UI-only property allowlist (`markdownDescription`, `aria-label`, etc.).

## Test plan

- [ ] `node index.js --action=restore` on a patched Cursor, confirm Tab works in English baseline
- [ ] Run localization on a test Cursor install (or copy of `workbench.desktop.main.js`)
- [ ] Verify `workbench.desktop.main.js` still contains `[..., "Tab", ..., "VK_TAB"]` (not `"Tab 补全"` in scan table)
- [ ] Enable **Keyboard Shortcuts Troubleshooting**, press Tab in editor → should see `Resolving Tab` / accept suggestion, **not** `cannot be dispatched`
- [ ] Settings sidebar still shows **Tab 补全** (or equivalent) for the Tab settings section

## Notes

Restoring English via `--action=restore` immediately fixes the issue for affected users; this PR prevents recurrence on re-localization.

EOF
)"
```

命令成功后会打印 PR URL，贴到浏览器即可。

**无 fork 时** `--head` 用 `Caph-dev:分支名`；若 push 到 `origin` 自己的 fork 远程名不同，改成 `git push -u fork 分支` 对应的 `owner:branch`。

---

## 步骤 8：PR 创建后

1. 在 PR 页面勾选 **Allow edits from maintainers**（若有）。
2. 如维护者要求，可附 `renderer.log` 片段或修复前后 grep 对比：

```bash
# 汉化后应仍为 "Tab"（扫描表），不应为 Tab 补全
grep -o '\[1,49,"[^"]*"' /Applications/Cursor.app/Contents/Resources/app/out/vs/workbench/workbench.desktop.main.js | head
```

3. **不要**在 PR 里提交 `.DS_Store`；可加 `.gitignore` 另 PR 或本地忽略。

---

## 给下一窗口 Agent 的速查

```bash
cd ~/Projects/cursor-i18n-tool
git status
git diff src/dict.js src/i18n-core.js
# 按本文「步骤 1–7」执行；优先聚焦 Tab 修复
```

---

## 附录 A：提交全部本地改动（一个大 PR）

```bash
git checkout -b feat/localization-dict-and-tab-fix
git add src/dict.js src/i18n-core.js
git commit -m "feat: expand dict and fix Tab keybinding scan table regression"
git push -u fork feat/localization-dict-and-tab-fix
gh pr create --repo Wuyf5275/cursor-i18n-tool --base main \
  --title "feat: dictionary updates + fix Tab keybinding i18n regression" \
  --body "..."  # 分段写 Summary / Tab fix / Dict expansion / Test plan
```

---

## 附录 B：上游可能问的点

| 问题 | 回答要点 |
|------|----------|
| 为何不从 `riskyShortWords` 修 regex 而删 Tab？ | 扫描表与 JSX 文本节点结构相同；删词 + scoped 替换更稳 |
| `isProtectedKeybindingContext` 会漏翻 UI 吗？ | 只保护 `VK_*`/扫描表附近；正常 `title:"Tab"` 仍可翻 |
| 其它短词（Enter、Shift）？ | 当前未在 risky 列表；guard 对未来同类词也有效 |
| 官方语言包？ | 本工具 patch `workbench.desktop.main.js`；与 `vscode-language-pack-zh-hans` 不同路径 |

---

## 附录 C：相关路径

| 用途 | 路径 |
|------|------|
| 本 handoff | `~/Projects/cursor-i18n-tool/PR-HANDOFF.md` |
| 原问题交接（课程仓库） | `~/Projects/Python-Core-50-Courses/.cursor-tab-i18n-handoff.md` |
| 被 patch 的文件 | `/Applications/Cursor.app/.../workbench.desktop.main.js` |

---

## 建议使用的 Skills（在 cursor-i18n-tool 窗口）

| Skill | 用途 |
|-------|------|
| `handoff` | 继续本任务 |
| `creating-pull-requests` | 按用户规则跑 git/gh |
| `find-docs` | `gh pr create` 参数查文档 |

---

**下一 agent：** 在 `~/Projects/cursor-i18n-tool` 按步骤 1–7 创建 fork、分支、聚焦提交、push、`gh pr create`；PR 正文可直接复制上文 `--body` 块。
