# Git 提交操作记录

> 记录将「Obsidian_to_Anki 笔记 ID 格式修改」提交并推送到 GitHub 的完整过程。

---

## 一、仓库信息

| 项目 | 值 |
|---|---|
| 仓库路径 | `/Users/reixu/VibeCoding/Obsidian_And_Anki` |
| 远程仓库 | `git@github.com:IOSure/Obsidian_And_Anki.git`（origin） |
| 分支 | `master` |
| Git 身份 | `IOSure <shuomaoembed@gmail.com>` |

---

## 二、提交内容

本次提交包含 2 个文件，共 **166 行新增、24 行删除**：

| 文件 | 状态 | 说明 |
|---|---|---|
| `obsidian_to_anki.py` | 修改（71 行变更） | 笔记 ID 标记从 `ID: <数字>` 改为可点击的 Anki 链接格式，并兼容旧格式读取 |
| `ID格式修改说明.md` | 新增（119 行） | 详述代码改动与验证过程 |

**未纳入本次提交**（本仓库其他任务的产物，与本次无关）：
`archify-architecture.json`、`architecture.html`、`architecture.visual-check.*` 等 8 个文件。

---

## 三、操作步骤与命令

### 1. 查看状态
```bash
git status
# 确认：obsidian_to_anki.py 已修改；ID格式修改说明.md 未跟踪；architecture.* 未跟踪
git remote -v      # 远程为 git@github.com:IOSure/Obsidian_And_Anki.git
git branch --show-current   # master
```

### 2. 暂存本次改动的文件（只加相关文件）
```bash
git add obsidian_to_anki.py "ID格式修改说明.md"
git status   # 两个文件进入 "Changes to be committed"
```

### 3. 提交
```bash
git commit -m "Change note ID marker to clickable Anki reference link
- Write '*Anki Reference: [Card <id>](anki://x-callback-url/search?query=cid:<id>)*'
  back into notes instead of 'ID: <id>'.
- Reader still recognises the legacy 'ID:' / '<!--ID: -->' markers so
  previously-synced cards are not duplicated.
- Update all parsing sites (Note, InlineNote, RegexNote, empty-note and
  fix-newline regexes, delete-id collection) via ID_MARKER / _id_from_match.
- Add ID格式修改说明.md describing the change and its verification."
```

### 4. 推送到 GitHub
```bash
git push origin master
# 输出：feb3db2..981a107  master -> master
```

---

## 四、提交详情

- **完整哈希**：`981a10725d44f8e6cfd84774da2e55d46d53cba1`
- **短哈希**：`981a107`
- **作者**：IOSure <shuomaoembed@gmail.com>
- **时间**：Thu Sep 24 12:47:53 2026 +0800
- **说明**：Change note ID marker to clickable Anki reference link

```text
commit 981a10725d44f8e6cfd84774da2e55d46d53cba1 (HEAD -> master, origin/master)
 2 files changed, 166 insertions(+), 24 deletions(-)
 create mode 100644 "ID格式修改说明.md"
```

---

## 五、验证结果

| 检查项 | 结果 |
|---|---|
| 推送输出 | `To github.com:IOSure/Obsidian_And_Anki.git`，`feb3db2..981a107 master -> master` ✅ |
| 本地分支与远程同步 | `Your branch is up to date with 'origin/master'` ✅ |
| 工作区 | 仅剩无关的 `architecture.*` 未跟踪文件，无遗留未提交的本次改动 ✅ |

---

## 六、备注

- 推送使用 SSH 方式（`git@github.com:...`），无需输入凭证即成功。
- `architecture.*` 系列文件与本任务无关，未纳入提交；如需清理可另行处理。

---

## 七、后续提交与发布操作

### 1. 版本升级提交 `2ad54d5`（manifest.json / versions.json）

发布前按 Obsidian 插件规则手动改版本文件并**先于打标签提交**：

| 文件 | 改动 |
|---|---|
| `manifest.json` | `version`：`3.6.0 → 3.7.0` |
| `versions.json` | 顶部新增 `"3.7.0": "0.9.20"` |

```bash
git add manifest.json versions.json
git commit -m "Bump version to 3.7.0"
git push origin master    # 8f8e3f2 之前、2ad54d5 推送
```
> 网页端创建 Release 不会自动改这两个文件，必须手动改并先提交。

### 2. TS 源码镜像提交 `8f8e3f2`（src/note.ts、src/file.ts、package-lock.json）

把 Python 端的 ID 新格式改动镜像进 TS 插件源码（否则发布构建出的 `main.js` 仍是旧格式）。提交前先本地验证：

```bash
npm install          # 本地原本无 node_modules
npm run build        # rollup 构建，created main.js in 1.5s
grep -c "Anki Reference" main.js        # 6
grep -c "anki://x-callback-url" main.js # 2
```
再用 Node 复刻正则做功能测试（新/旧/注释/带空格/无 ID/删除采集/RegexNote 分组）全部通过，然后：

```bash
git add src/note.ts src/file.ts package-lock.json
git commit -m "Mirror clickable Anki reference link to TS plugin source (src/)"
git push origin master   # 8f8e3f2..推送
```

### 3. 发布形态：draft → 直接发布 `c164dce`

参考仓库 `ObsidianToAnki/Obsidian_to_Anki` 是**自动直接发布**，而本仓库工作流 `draft: true`。改为 `false`：

```bash
# 编辑 .github/workflows/obsidian-release.yml: draft: true → draft: false
git add .github/workflows/obsidian-release.yml
git commit -m "Publish releases directly instead of drafts"
git push origin master   # 8f8e3f2..c164dce master -> master
```

### 4. 标签 `v3.7.0` 重指向（两次）

`v3.7.0` 需指向「包含 TS 改动 + draft:false 工作流」的最终提交 `c164dce`，因此重打标签并推送（触发发布工作流重建 `main.js` 并自动发布）：

```bash
git tag -d v3.7.0
git push origin :refs/tags/v3.7.0        # 删除远端标签
git tag v3.7.0 c164dce
git push origin v3.7.0                    # 重新推送，触发 Actions
git rev-list -n1 v3.7.0                   # c164dce…（确认指向正确）
```

### 5. 发布结果

| 项目 | 值 |
|---|---|
| 触发方式 | `push tags v3.7.0`（push 到 `c164dce`） |
| 工作流 | `.github/workflows/obsidian-release.yml` |
| 构建 | `npm install obsidian` → `npm install` → `npm ci` → `npm run build` |
| 打包 | `main.js`、`manifest.json`、`styles.css`、`README.md` → `obsidian-to-anki-plugin-3.7.0.zip` |
| Release 资产 | `main.js`、`manifest.json`、`styles.css`、`obsidian-to-anki-plugin-3.7.0.zip`、Source code(zip/tar.gz) |
| 发布 | `draft: false`，自动生成 What's Changed / Full Changelog，直接发布正式 Release |

> `main.js` 由工作流从 `src/` 构建，不入 git；本次 Release 里的 `main.js` 是新 ID 链接格式。

### 6. 发布遇到的两个问题与修复

| 问题 | 原因 | 修复 |
|---|---|---|
| 首次手动 dispatch 运行在 **Release 步失败** | 工作流缺 `permissions`，`softprops/action-gh-release` 无 `contents: write` 权限无法建 Release | 工作流顶部新增 `permissions: contents: write`（提交 `ce29da9`） |
| 标签推送曾未触发运行 | 事件/时序问题 | 重新 `git tag -d v3.7.0` → 删远端标签 → 重建到最新提交 → 推送，触发 Actions |

### 7. 最终发布验证（GitHub Actions + 下载校验）

- 触发：`push tags v3.7.0` → 运行成功（Build / Package / **Release 全部 success，run 已完成）。
- Release：`v3.7.0`，`draft:false`、非预发布，`published_at 2026-09-26T15:33:34Z`。
- 资产（4 个）：`main.js`、`manifest.json`、`styles.css`、`obsidian-to-anki-plugin-v3.7.0.zip`，自动生成 Full Changelog。
- 下载发布包 `main.js`（1962713 字节）校验：`Anki Reference`×6、`anki://x-callback-url/search?query=cid:`×2、旧写入 0 处 → **新 ID 链接格式确认**。
- 地址：https://github.com/IOSure/Obsidian_And_Anki/releases/tag/v3.7.0


