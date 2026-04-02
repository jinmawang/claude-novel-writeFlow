# Context 追踪系统实现计划

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 为写作工作流新增 `context/` 上下文追踪系统，确保 100 章长篇小说写作过程中人物、世界设定、关键事件的连续性一致。

**Architecture:** 新增 `/context` 命令用于初始化和重建上下文；在 `/write` 流程中嵌入两个新环节——写作前读取 context 文件传给 Agent，保存后自动运行 Context Extractor Agent 提取新事实（含去重和更新逻辑）；同步升级 Continuity Reviewer 的检查项。

**Tech Stack:** Claude Code 自定义斜杠命令（Markdown prompt），Agent 工具（子智能体），Read/Write/Bash 工具。

---

## 五个问题的解决方案对照

| 问题 | 解决方案 | 对应任务 |
|------|----------|----------|
| 存量章节无法初始化 | 新建 `/context` 命令，批量分析已有章节 | Task 1 |
| 只能追加不能更新 | Extractor Agent 提示明确：先读现有内容，对变更条目执行更新而非追加 | Task 4 |
| 去重逻辑缺失 | Extractor Agent 运行前读取当前 context，逐条对比后只写入真正新增/变更内容 | Task 4 |
| Continuity Reviewer 未接入 context | 向 Continuity Reviewer 传入 context 文件 + 新增第12项检查 | Task 3 |
| Extractor 提取边界不清 | 明确规则：只提取正文散文中具体出现的描写，大纲中已有的计划内容不算已确立事实 | Task 4 |

---

## Task 1：新建 /context 命令

**Files:**
- Create: `.claude/commands/context.md`

**Step 1: 创建命令文件**

文件完整内容如下：

```markdown
---
description: 初始化或重建 context/ 上下文追踪文件，从已有章节中提取已确立事实。
argument-hint: [--init] [--rebuild]
allowed-tools: Read, Write, Bash, Agent
---

你是上下文管理助手。负责创建和维护 context/ 文件夹中的四个追踪文件。

用户输入参数：$ARGUMENTS

---

## 第一步：环境检查

- 已写章节：!`ls chapters/chapter-*.md 2>/dev/null | sort || echo "无"`
- context 目录：!`test -d context && echo "EXISTS" || echo "MISSING"`
- characters.md：!`test -f context/characters.md && echo "EXISTS" || echo "MISSING"`
- world.md：!`test -f context/world.md && echo "EXISTS" || echo "MISSING"`
- continuity.md：!`test -f context/continuity.md && echo "EXISTS" || echo "MISSING"`
- timeline.md：!`test -f context/timeline.md && echo "EXISTS" || echo "MISSING"`

---

## 第二步：模式判断

- 包含 `--init`，或 context/ 不存在，或所有 context 文件均不存在 → 进入【初始化模式】
- 包含 `--rebuild` → 进入【重建模式】
- 无参数且 context 文件已存在且有已写章节 → 进入【重建模式】

---

## 【初始化模式】

用 Bash 创建目录和四个空模板文件：

```bash
mkdir -p context
```

依次用 Write 工具创建以下四个文件（若文件已存在则跳过）：

**`context/characters.md`**
```markdown
# 人物档案

> 记录正文中已确立的人物细节（外貌、口头禅、伤情、能力变化等）。
> 格式：`- 条目描述 \`chXX\``
> 规则：仅记录正文散文中具体写出的内容，大纲中的计划内容不在此列。

```

**`context/world.md`**
```markdown
# 世界设定

> 记录正文中已出现的地名、规则、道具、势力、组织等设定细节。
> 格式：`- 条目描述 \`chXX\``

```

**`context/continuity.md`**
```markdown
# 关键事件

> 记录正文中发生的重大事件，供后续章节保持一致性。
> 格式：`- [chXX] 事件描述`

```

**`context/timeline.md`**
```markdown
# 故事时间线

> 记录故事内的时间推进，避免时间矛盾。
> 格式：`- [chXX] 故事内时间点描述`

```

若无已写章节：告知用户"context/ 已初始化，开始写作后每章完成时将自动更新。"，停止执行。

若有已写章节：直接进入【重建模式】的"分析阶段"。

---

## 【重建模式】

### 准备阶段

若 context 文件不存在，先执行【初始化模式】创建空模板文件。

若包含 `--rebuild`，清空四个文件内容（保留文件头部的说明注释）。

告知用户："正在分析已有章节，重建上下文档案……"

### 分析阶段

读取所有已写章节文件列表，按章节编号排序。

**逐章分析：** 对每个章节文件，启动一个 **Context Extractor Agent**，提供以下提示：

```
你是上下文提取智能体。从以下章节中提取"已确立事实"，更新到 context 文件中。

【第N章正文】
[章节完整内容]

【当前 context 文件内容】
characters.md：[当前内容]
world.md：[当前内容]
continuity.md：[当前内容]
timeline.md：[当前内容]

【提取规则】
1. 只提取正文散文中具体描写出来的细节，大纲式的"计划内容"不算已确立事实
2. 对照现有 context 内容，跳过已记录的条目（去重）
3. 若本章对已有条目产生变化（如伤情痊愈、状态改变），修改原条目而非新增重复条目
   - 修改格式：在原条目后追加变化描述，例如：
     `- 右臂骨折 \`ch45\` → 痊愈 \`ch52\``
4. 严格按现有文件格式追加，不改变文件头部说明

【输出格式】
分别输出四个文件需要新增或修改的内容：

characters.md 变更：
[新增行或修改说明，若无则写"无"]

world.md 变更：
[新增行或修改说明，若无则写"无"]

continuity.md 变更：
[新增行或修改说明，若无则写"无"]

timeline.md 变更：
[新增行或修改说明，若无则写"无"]
```

收到 Extractor Agent 输出后，用 Write/Edit 工具将变更追加或修改到对应文件。

### 完成报告

所有章节分析完毕后，告知用户：
"✓ context/ 重建完成，共分析 X 章，更新记录如下：
- characters.md：X 个人物，X 条记录
- world.md：X 个设定条目
- continuity.md：X 个关键事件
- timeline.md：X 个时间节点"

---

## 错误处理

- 若无任何已写章节且无 `--init` 参数：告知用户"无已写章节可分析，已创建空 context 文件。开始写作后将自动更新。"
- 若某章文件读取失败：跳过该章，完成后在报告中注明
```

**Step 2: 验证**

通读文件，确认：
- 三种模式（init/rebuild/无参数）的判断逻辑清晰
- Extractor Agent 提示包含去重规则、更新规则、提取边界规则
- 输出格式明确

**Step 3: Commit**

```bash
cd /Users/fengzhongjincao/Documents/aidemo/claude-novel-writeFlow
git add .claude/commands/context.md
git commit -m "feat: 新增 /context 命令，支持初始化和重建上下文追踪文件"
```

---

## Task 2：write.md — Step 1 环境检查加入 context 状态

**Files:**
- Modify: `.claude/commands/write.md`（第一步，约第13-21行）

**Step 1: 在环境检查末尾追加**

找到第一步环境检查的末尾（`- 大纲章节文件：...` 那行之后），追加：

```markdown
- context 目录：!`test -d context && echo "EXISTS" || echo "MISSING"`
- context 文件状态：!`for f in context/characters.md context/world.md context/continuity.md context/timeline.md; do test -f "$f" && echo "$f: EXISTS" || echo "$f: MISSING"; done 2>/dev/null || echo "context/ 目录不存在"`
```

**Step 2: 验证**

第一步环境检查现在共检查 7 项（原 5 项 + context 目录 + context 文件状态）。

**Step 3: Commit**

```bash
git add .claude/commands/write.md
git commit -m "feat: write.md 环境检查新增 context 目录状态"
```

---

## Task 3：write.md — Step 5 读取 context 文件

**Files:**
- Modify: `.claude/commands/write.md`（第五步，约第75-101行）

**Step 1: 在第五步读取列表末尾追加4项**

找到第五步读取列表（以 `9. chapters/chapter-(N-2).md` 结尾），在其后、`> 路径格式` 注释之前追加：

```markdown
10. `context/characters.md`（人物档案，若存在）
11. `context/world.md`（世界设定，若存在）
12. `context/continuity.md`（关键事件，若存在）
13. `context/timeline.md`（故事时间线，若存在）
```

**Step 2: 在"上下文长度控制"中补充 context 文件的处理规则**

在现有截断规则末尾追加：

```markdown
- context 文件处理：四个文件全量传入（预计总量 < 5000 字，无需截断）。若单个文件超过 3000 字，优先保留与本章大纲中出现的人名/地名相关的条目。
```

**Step 3: 验证**

第五步读取列表现在共 13 项，截断规则覆盖 context 文件。

**Step 4: Commit**

```bash
git add .claude/commands/write.md
git commit -m "feat: write.md 第五步新增读取 context 四个文件"
```

---

## Task 4：write.md — Step 6 Writer Agent 和 Continuity Reviewer 接入 context

**Files:**
- Modify: `.claude/commands/write.md`（第六步，Writer Agent 提示和 Continuity Reviewer 提示）

**Step 1: Writer Agent 提示中新增【已确立上下文】章节**

找到 Writer Agent 提示中`【风格规范】`章节之前，插入：

```
【已确立上下文（正文中已写出的事实，必须严格遵守）】

人物档案（characters.md）：
[粘贴 context/characters.md 内容，若不存在则写"暂无记录"]

世界设定（world.md）：
[粘贴 context/world.md 内容，若不存在则写"暂无记录"]

关键事件（continuity.md）：
[粘贴 context/continuity.md 内容，若不存在则写"暂无记录"]

故事时间线（timeline.md）：
[粘贴 context/timeline.md 内容，若不存在则写"暂无记录"]

```

**Step 2: Continuity Reviewer 提示中新增 context 文件**

找到 Continuity Reviewer 提示中`【整体大纲概要】`章节之前，插入：

```
【已确立上下文】

人物档案：[粘贴 context/characters.md 内容，若不存在则写"暂无记录"]
世界设定：[粘贴 context/world.md 内容，若不存在则写"暂无记录"]
关键事件：[粘贴 context/continuity.md 内容，若不存在则写"暂无记录"]
时间线：[粘贴 context/timeline.md 内容，若不存在则写"暂无记录"]

```

**Step 3: Continuity Reviewer 检查项末尾新增第12项**

在"整体大纲一致性"章节的第11条之后，新增：

```
**已确立事实一致性：**
12. 本章中人物的外貌、口头禅、伤情、能力等描写是否与 characters.md 记录一致？
13. 本章出现的地名、规则、道具等是否与 world.md 记录一致？
14. 本章涉及的历史事件是否与 continuity.md 记录一致？时间推进是否符合 timeline.md？
```

**Step 4: 验证**

- Writer Agent 提示有【已确立上下文】章节
- Continuity Reviewer 提示有【已确立上下文】章节
- Continuity Reviewer 检查项从 11 条扩展为 14 条，新增"已确立事实一致性"维度

**Step 5: Commit**

```bash
git add .claude/commands/write.md
git commit -m "feat: write.md Writer Agent 和 Continuity Reviewer 接入 context 文件"
```

---

## Task 5：write.md — 新增 Step 7.5 Context Extractor Agent

**Files:**
- Modify: `.claude/commands/write.md`（第七步之后，第八步之前）

**Step 1: 插入完整的 Step 7.5**

在第七步"保存章节"末尾（"保存成功后告知用户：..."那段）之后、第八步"自动模式循环"之前，插入：

```markdown
---

## 第 7.5 步：提取上下文（Context Extraction）

**无论是否使用 --no-review，此步骤均执行。**

### 初始化检查

若 `context/` 目录不存在，先自动创建目录和四个空模板文件：

```bash
mkdir -p context
```

依次检查并创建（若不存在）：
- `context/characters.md`：写入标准模板头部
- `context/world.md`：写入标准模板头部
- `context/continuity.md`：写入标准模板头部
- `context/timeline.md`：写入标准模板头部

（模板头部格式见 `/context` 命令中的初始化内容）

### 启动 Context Extractor Agent

使用 Agent 工具启动 **Context Extractor Agent**，提供以下提示：

```
你是上下文提取智能体。从刚完成的章节中提取"已确立事实"，更新到 context 文件中。

【第N章正文（刚写完的章节）】
[粘贴刚保存的章节完整内容]

【当前 context 文件内容（提取前的状态）】

characters.md 当前内容：
[粘贴 context/characters.md 完整内容]

world.md 当前内容：
[粘贴 context/world.md 完整内容]

continuity.md 当前内容：
[粘贴 context/continuity.md 完整内容]

timeline.md 当前内容：
[粘贴 context/timeline.md 完整内容]

【提取规则（严格遵守）】
1. 只提取正文散文中具体描写出来的细节——人物外貌、口头禅、伤情、地点细节、规则描述、事件发生等
2. 大纲中已有的计划性内容不算已确立事实；只有正文写出来才算
3. 对照现有 context 内容逐条比对，已有记录的条目直接跳过（去重）
4. 若本章对已有条目产生变化（伤情痊愈、状态改变、关系转变），在原条目末尾追加变化记录：
   格式：`- 右臂骨折 \`ch45\` → 痊愈 \`ch52\``
5. 不删除已有条目，只新增或追加变化
6. 保持现有文件的格式和缩进，不修改文件头部说明行

【输出格式】
对每个需要变更的文件，输出完整的新文件内容（包含原有内容 + 本次新增/修改）。
若某个文件无需变更，写"[文件名]：无变更"。

characters.md 新内容：
[完整文件内容或"无变更"]

world.md 新内容：
[完整文件内容或"无变更"]

continuity.md 新内容：
[完整文件内容或"无变更"]

timeline.md 新内容：
[完整文件内容或"无变更"]
```

### 应用提取结果

收到 Context Extractor Agent 输出后：
- 对每个"有新内容"的文件，用 Write 工具覆盖写入新内容
- 对"无变更"的文件，跳过
- 告知用户："✓ 第 N 章上下文已更新"（简短提示，不展示详细变更内容）

### 错误处理

- 若 Context Extractor Agent 调用失败：告知用户"上下文提取失败，章节已保存。可稍后运行 `/context` 手动更新。"，继续执行后续步骤，不中断写作流程
```

**Step 2: 验证**

确认：
- Step 7.5 位于第七步之后、第八步之前
- 注明"无论是否使用 --no-review，此步骤均执行"
- 包含初始化检查（首次运行时自动创建 context 文件）
- Extractor Agent 提示包含完整的5条提取规则
- 失败时不中断写作流程（容错）

**Step 3: Commit**

```bash
git add .claude/commands/write.md
git commit -m "feat: write.md 新增 Step 7.5 Context Extractor Agent，含去重和更新逻辑"
```

---

## Task 6：更新 README.md

**Files:**
- Modify: `README.md`

**Step 1: 项目文件结构中新增 context/**

找到文件结构代码块，在 `style-rules.md` 行之后新增：

```
├── context/                    # 上下文追踪（/write 自动维护）
│   ├── characters.md           # 已确立人物细节
│   ├── world.md                # 已确立世界设定
│   ├── continuity.md           # 关键事件记录
│   └── timeline.md             # 故事内时间线
│
```

**Step 2: 命令详解中新增 /context 说明**

在 `/status` 命令说明之前，新增：

```markdown
### `/context` — 上下文管理

```
/context [--init] [--rebuild]
```

| 参数 | 说明 |
|------|------|
| 无参数 | 自动模式：若 context 文件不存在则初始化，否则从已有章节重建 |
| `--init` | 仅创建空的 context 文件（不分析章节） |
| `--rebuild` | 清空并从全部已有章节重新提取，适合中途引入本功能时使用 |

**日常使用无需手动运行**——每次 `/write` 完成后自动提取当章新增事实。

以下情况需要手动运行：
- 项目已有多章内容，首次引入 context 功能时：运行 `/context` 初始化
- 发现 context 记录有误需要重建：运行 `/context --rebuild`

---
```

**Step 3: 使用技巧中新增 context 相关说明**

在"使用技巧"末尾追加：

```markdown
**关于上下文追踪**
`context/` 文件夹由 `/write` 自动维护，记录正文中已确立的人物细节、世界设定、关键事件和时间线。Continuity Reviewer 会在每章审核时比对这些记录，防止前后矛盾。如果中途引入该功能，记得先运行 `/context` 初始化存量章节的上下文。
```

**Step 4: Commit**

```bash
git add README.md
git commit -m "docs: README 新增 context/ 结构说明和 /context 命令文档"
```

---

## 执行顺序

Tasks 1-6 存在依赖关系，需顺序执行：

```
Task 1（新建/context命令）
  ↓
Task 2（write.md 环境检查）
  ↓
Task 3（write.md 读取 context）
  ↓
Task 4（write.md Agent 提示接入 context）
  ↓
Task 5（write.md Step 7.5）
  ↓
Task 6（README）
```

Tasks 2-5 均修改同一个文件（write.md），必须串行执行，每步 commit 后再开始下一步。
