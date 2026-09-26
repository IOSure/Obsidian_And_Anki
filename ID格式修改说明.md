# Obsidian_to_Anki 笔记 ID 格式修改说明

> 目标：将插件写回笔记的 ID 标记，从旧的 `ID: <数字>` 改为可点击的 Anki 链接格式：
>
> `*Anki Reference: [Card 1765379536358](anki://x-callback-url/search?query=cid:1765379536358)*`

---

## 一、目标与设计思路

目标格式：`*Anki Reference: [Card 1765379536358](anki://x-callback-url/search?query=cid:1765379536358)*`

关键点：**这个标记里卡片 ID 出现了两次**（`[Card N]` 和 `query=cid:N`），而插件原有逻辑到处用「单个捕获组 `(\d+)`」来读 ID。所以改动分两部分：

- **写入端**：只输出新链接格式；
- **读取端**：必须能识别新格式，同时保留对旧格式 `ID: <数字>` / `<!--ID: <数字>-->` 的兼容——否则已经同步过的卡片会被当成新卡**重复添加**。

实现上，新格式的正则要有 `(?P=_anki_new_id)` 这种**反向引用**来保证两次出现的数字一致；为了读取端「一个 ID 只占一组捕获位」，先试了两种办法都被环境限制卡住（Python 3.14 不允许同名命名组 `(?P<id>)` 复用；分支重置组 `(?|...)` 此环境不支持），最终采用**两个独立命名组**（`_anki_new_id` / `_anki_legacy_id`），再用统一的 `_id_from_match()` 从匹配里取其中一个。

---

## 二、代码改动明细（obsidian_to_anki.py）

### 1. 模块顶部新增常量与辅助函数（原 34 行附近）

```
ID_PREFIX            —— 保留，仅作为旧格式说明
ANKI_REF_LINK        = r"\*Anki Reference: \[Card (?P<_anki_new_id>\d+)\]"
                       r"\(anki://x-callback-url/search\?query=cid:(?P=_anki_new_id)\)\*"
LEGACY_ID_LINK       = r"(?:<!--[ \t]*)?ID: (?P<_anki_legacy_id>\d+)"
ID_MARKER            = r"(?:" + ANKI_REF_LINK + r"|" + LEGACY_ID_LINK + r")"
ID_LINK_REGEXP_STR   = r"\n?(?:" + ANKI_REF_LINK + r"|" + LEGACY_ID_LINK + r".*)"
_id_from_match(match) —— 从任一命名组取出 int(ID)
```

`LEGACY_ID_LINK` 里 `(?:<!--[ \t]*)?` 的 `[ \t]*` 是为了同时容忍 `<!--ID: 123-->`（插件旧版自己写的样子）和手工写的 `<!-- ID: 123 -->`。

### 2. 块笔记 `Note`（原 502–517 行）

- `ID_REGEXP`：`r"(?:<!--)?" + ID_PREFIX + r"(\d+)"` → `re.compile(ID_MARKER)`
- `__init__`：原来是 `match` 两次、`int(...group(1))`；改为先取 `id_match` 一次 → `_id_from_match(id_match)` → `self.lines.pop()`。

### 3. 行内笔记 `InlineNote`（原 578–590 行）

- `ID_REGEXP`：同上改为 `re.compile(ID_MARKER)`
- `__init__`：`int(ID.group(1))` → `_id_from_match(ID)`。

### 4. 正则笔记 `RegexNote`（原 627–639 行）——最关键的一处

- `ID_REGEXP_STR`：从「**单个**尾部捕获组 `r"\n?(?:<!--)?(?:ID: (\d+).*)"`」改为 `ID_LINK_REGEXP_STR`（产出**两个**尾部组：新格式一组、旧格式一组）。
- `__init__` 的 id 分支：原来 `self.identifier = int(self.groups.pop())`（弹掉最后一组）；改为

  ```python
  self.identifier = _id_from_match(self.match)
  self.groups = self.groups[:-2]   # 丢弃尾部两个 ID 组，剩余才是字段组
  ```

  因为 `self.groups` 是按位置 zip 到字段上的，必须把多出来的 ID 组去掉，字段/标签映射才不错位。

### 5. 空笔记删除相关正则

- `App.EMPTY_REGEXP`（原 1102–1104）：`r"\n(?:<!--)?" + ID_PREFIX` → `r"\n" + ID_MARKER`
- `App.INLINE_EMPTY_REGEXP`（原 1135）：`r"\s+(?:<!--)?" + ID_PREFIX + r".*?"` → `r"\s+" + ID_MARKER + r".*?"`
- `RegexFile.fix_newline_ids`（原 1582）：`(\r\n|\r|\n){2}(?:<!--)?" + ID_PREFIX + r"\d+"` → `(\r\n|\r|\n){2}" + ID_MARKER`

### 6. 删除卡片 ID 采集（两处，`File.scan_file` 和 `RegexFile.scan_file`）

原 `int(match.group(1))` → `_id_from_match(match)`。这两处结构完全相同，用 `replace_all` 一次替换。

### 7. 写入端 `File.id_to_str`（原 1306–1315 行）

```python
result = (
    "*Anki Reference: [Card {id}]"
    "(anki://x-callback-url/search?query=cid:{id})*".format(id=id)
)
if inline: result += " "   else: result += "\n"
```

`comment` 参数保留但不再生效（新标记是可见链接，不能再用 `<!-- -->` 包起来）。

---

## 三、验证过程

### 第 1 步：语法编译

`python3 -m py_compile obsidian_to_anki.py` → 输出 `COMPILE OK`，exit 0，确认无语法错误。

### 第 2 步：功能测试（导入真实模块）

导入时需要把 import 期就构造 `markdown.Markdown` 的 `markdown` 依赖做一个最小 stub（环境里没有真正装），其余全部走真实代码；再手工设好 `App.FIELDS_DICT` 和 `CONFIG_DATA["CurlyCloze"]`。逐项验证：

| 场景 | 期望 | 实测 |
|---|---|---|
| 块笔记，末行是新链接 | id=1765379536358，且该行被移除 | ✅ |
| 块笔记，旧格式明文 `ID: 1786956984681` | id=1786956984681 | ✅ |
| 块笔记，旧格式注释 `<!--ID: 1786956984681-->`（无空格） | 解析成功 | ✅ |
| 块笔记，旧格式注释 `<!-- ID: … -->`（有空格） | 解析成功 | ✅ |
| 块笔记无 ID | identifier=None | ✅ |
| 行内笔记新链接 / 旧格式 | 都能取到 id，正文前缀保留 | ✅ |
| `id_to_str` 块 / 行内 / 带 comment | 输出与目标格式逐字符一致，comment 被忽略 | ✅ |
| RegexNote：字段+标签+新链接 ID 组合提取 | id=42，tags=[t1,t2]，字段正确 | ✅ |
| RegexNote：字段+标签+旧格式 ID | id=77 | ✅ |
| RegexNote 无 ID（走「新增」分支） | identifier=None | ✅ |

### 第 3 步：删除/清理路径专项测试

- 用 `re.escape("<!--DELETE-->") + RegexNote.ID_REGEXP_STR` 模拟 `RegexFile.EMPTY_REGEXP`，分别喂新链接、`ID: 123`、`<!--ID: 123-->` → 均正确提取删除用 id。
- `fix_newline_ids` 的 `{2}`+`ID_MARKER` 对两种格式 → 正确把双换行收敛成单换行。
- `App.EMPTY_REGEXP` 片段对两种格式 → 均能匹配空笔记块。

---

## 四、已知边界（未改动，供决策）

1. 仓库 `tests/` 里的单元/e2e 测试各自写死了旧的 `ID:` 正则，它们需要真实 Anki 运行时才能跑，本次未改、未验证；
2. README 里关于 ID 格式的说明仍是旧版；
3. 已按旧格式同步过的笔记读取兼容（不会重复建卡），但标记文字会保留旧样，直到该笔记被再次写入——需要的话可批量替换成新格式。

---

## 五、TS 插件源码（src/）镜像改动

桌面脚本 `obsidian_to_anki.py` 改完后，为确保发布工作流构建出的 `main.js` 也是新格式，把同一套逻辑镜像进 TypeScript 插件源码。`main.js` 不在 git 仓库（`.gitignore` 排除），由工作流每次 `npm run build` 从 `src/` 生成——所以**必须改 `src/`，否则发布出来的插件写回笔记仍是旧格式**。

### 1. `src/note.ts`（读取端）

- 顶部：`ID_REGEXP_STR` 从单个尾部捕获组改为**两个分支、两个命名组**：

  ```
  ANKI_REF_LINK  = \*Anki Reference: \[Card (?<_anki_new_id>\d+)\]
                   \(anki://x-callback-url/search\?query=cid:\k<_anki_new_id>\)\*
  LEGACY_ID_LINK = (?:<!--[ \t]*)?ID: (?<_anki_legacy_id>\d+)
  ID_REGEXP_STR  = \n?(?:ANKI_REF_LINK|LEGACY_ID_LINK.*)
  ```

  JS 命名组用 `(?<name>…)`，反向引用用 `\k<name>`（Python 是 `(?P=name)`）。新分支 ID 出现两次，靠 `\k<_anki_new_id>` 保证 `[Card N]` 与 `query=cid:N` 的数字一致。
- 新增导出 `get_id_from_match(match)`：读 `match.groups`，取 `_anki_new_id` 或 `_anki_legacy_id` 之一返回 `number | null`。
- `Note.ID_REGEXP`（实例字段）与 `InlineNote.ID_REGEXP`（static）：从 `/(?:<!--)?ID: (\d+)/` 改为同时匹配两种格式的组合正则。
- `Note.getIdentifier()` / `InlineNote.getIdentifier()`：原来 `int(result[1])` → `get_id_from_match(result)`。
- `RegexNote` 构造的 id 分支：原来 `parseInt(this.match.pop())`（弹一组）；现在
  ```ts
  this.identifier = get_id_from_match(match)
  this.match.pop(); this.match.pop()   // 丢弃尾部两个 ID 组，剩余才是字段组
  ```
  因为 `ID_REGEXP_STR` 现在产出**两个**尾部捕获组（新、旧各一），按位置映射字段前必须弹掉，标签/字段才不会错位。

### 2. `src/file.ts`（写入端）

- `id_to_str()`：改为输出 `*Anki Reference: [Card N](anki://x-callback-url/search?query=cid:N)*`；`comment` 参数保留但忽略（新标记是可见链接，不能用 `<!-- -->` 包裹）。
- `double_regexp`：`(?:\r\n|\r|\n)((?:\r\n|\r|\n)(?:<!--)?ID: \d+)` → 同时匹配新链接与旧 `ID:` 两种格式，用于 `fix_newline_ids` 把双换行收敛成单换行。
- `scanDeletions()`：`parseInt(match[1])` → `get_id_from_match(match)`（EMPTY_REGEXP 现在有两个 ID 组，`match[1]` 不可靠）。

### 3. `src/setting-to-data.ts`（无需改动）

`EMPTY_REGEXP = new RegExp(escapeRegex(...) + ID_REGEXP_STR, "g")` 直接复用导出的 `ID_REGEXP_STR`，自动继承新格式。

### 4. TS 侧验证

| 检查项 | 结果 |
|---|---|
| `npm install` | 成功（本地原本无 node_modules） ✅ |
| `npm run build`（rollup） | 成功，`created main.js in 1.5s` ✅ |
| 生成的 `main.js` 含新格式 | `"Anki Reference"` 出现 6 处、`anki://x-callback-url/search?query=cid:` 2 处 ✅ |
| 生成的 `main.js` 无旧写入 | 旧 `let result = "ID: " + identifier` 为 0 处 ✅ |
| 功能测试（复刻正则，Node 实测） | 新链接→id、旧明文 `ID:` →id、注释 `<!--ID:-->`、带空格 `<!-- ID: -->`、无 ID→null，全部符合 ✅ |
| 删除采集（`matchAll` + EMPTY_REGEXP） | 新/旧/注释 4 种样本 id 均正确提取 ✅ |
| RegexNote 组合（字段+标签+ID） | 新链接 id=321、旧明文 id=456、旧注释 id=789，字段/标签分组正确 ✅ |


