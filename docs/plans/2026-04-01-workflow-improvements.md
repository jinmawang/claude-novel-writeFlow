# claude-novel-writeFlow 工作流完善计划

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 修复现有三个命令中的 bug 和缺失功能，并新增 `/export`、`/status` 两个实用命令，完善整体写作工作流。

**Architecture:** 所有命令均为 `.claude/commands/` 下的 Markdown 提示词文件，不涉及可执行代码。"测试"方式为人工验证命令逻辑的完整性与一致性（逐条检查触发条件、分支处理、输出格式）。

**Tech Stack:** Claude Code 自定义斜杠命令（Markdown prompt），Bash inline checks（`!` 语法），Agent 工具（多智能体并行）。

---

## 问题清单（现状分析）

### Bug
- **B1** `style.md` 第十节 AI味检测清单只有 7 条，但 README 和功能描述均说"8条"，少了一条
- **B2** `write.md` 第七步保存时说"超过99章时自动升级为三位数"，但第一步环境检查和第五步读取素材的路径均写死为两位数零填充，三位数章节会导致路径不一致

### 功能缺失
- **F1** 无 `/export` 命令：无法将所有已写章节合并为单一完整稿件文件
- **F2** 无 `/status` 命令：无法快速查看写作进度（已完成几章/共几章/总字数）
- **F3** `write.md` 无 `--no-review` 标志：每次写作强制走三智能体审核，粗稿阶段耗时且无必要
- **F4** `outline.md` 无"仅补充单章大纲"模式：当整体大纲已有但某章大纲缺失时，必须手动修改或重走完整交互流程
- **F5** `write.md` 无上下文截断策略：overview.md + style-rules.md + 4章大纲 + 2章正文同时注入 Writer Agent，大型项目（100+章/10万字大纲）可能超出上下文窗口

### 体验改善
- **E1** `write.md` `--auto` 模式每章都暂停确认，缺少 `--force` 子参数实现真正无干预连续写作

---

## Task 1：修复 style.md AI味检测清单（Bug B1）

**Files:**
- Modify: `.claude/commands/style.md`（第十节，补充第8条检测项）

**Step 1: 确认当前条目**

打开 `style.md` 数第十节已有的7条内容，确认缺失的是：
> 是否存在"终于""此刻""突然"等无意义的时间副词滥用？（频率过高会让节奏失真）

**Step 2: 在第7条后插入第8条**

在 `- [ ] 是否有过多因果连接词` 之后、`---` 分隔线之前，添加：
```markdown
- [ ] 是否滥用时间副词？（"终于""此刻""突然"频繁出现会破坏叙事节奏）
```

**Step 3: 人工验证**

通读第十节，确认共有8条检测项，编号与 README 描述一致。

**Step 4: Commit**

```bash
git add .claude/commands/style.md
git commit -m "fix: style.md AI味检测清单补充第8条，修复数量与README不一致"
```

---

## Task 2：修复 write.md 三位数章节路径不一致（Bug B2）

**Files:**
- Modify: `.claude/commands/write.md`

**Step 1: 定位三处涉及路径格式的描述**

在 `write.md` 中找到以下三处：
1. 第一步环境检查：`ls chapters/chapter-*.md`（无问题，通配符兼容）
2. 第五步读取素材：`outline/chapter-(N-2).md` 等描述中隐含两位数格式
3. 第七步保存：`chapters/chapter-NN.md（两位数零填充，如 chapter-01.md、chapter-12.md；超过99章时自动升级为三位数）`

**Step 2: 统一路径格式描述**

将第五步和第七步中涉及路径格式的说明修改为：

```markdown
文件名格式：章节号 ≤ 99 时两位数零填充（chapter-01.md），章节号 ≥ 100 时三位数（chapter-100.md）。
读取前序/后续章节大纲时，同样遵循此规则。
```

并在第五步的读取列表末尾添加一行备注：
```markdown
> 注意路径格式：章节号 < 100 用两位数（`chapter-01.md`），≥ 100 用三位数（`chapter-100.md`）
```

**Step 3: 人工验证**

检查第一步、第四步、第五步、第七步、第八步中所有涉及路径的地方，确认格式规则一致。

**Step 4: Commit**

```bash
git add .claude/commands/write.md
git commit -m "fix: write.md 三位数章节路径规则统一，消除两/三位数格式不一致"
```

---

## Task 3：write.md 新增 --no-review 标志（Feature F3）

**Files:**
- Modify: `.claude/commands/write.md`

**Step 1: 更新 argument-hint 和参数解析**

在文件头部 `argument-hint` 改为：
```
[章节号] [--words=3000] [--auto] [--no-review]
```

在第三步参数解析末尾，新增：
```markdown
- **快速模式**：检查是否包含 `--no-review`（跳过两个审核智能体，直接保存初稿）
```

在解析示例中补充：
```
- `/write 5 --no-review` → 章节5，3000字，跳过审核直接保存
```

**Step 2: 在第六步三智能体流程入口处添加分支**

在"### 轮次循环"之前插入：

```markdown
**快速模式分支（--no-review）：**
若解析到 `--no-review`，跳过审核流程：
1. 仅启动 Writer Agent，获取初稿
2. 告知用户："快速模式：跳过审核，直接保存初稿。"
3. 直接跳到【第七步：保存章节】

---
```

**Step 3: 人工验证**

通读第三步～第七步，确认：
- `--no-review` 下 Writer Agent 仍收到完整素材包
- 两个 Reviewer Agent 在此模式下不被启动
- 保存流程与正常模式完全一致

**Step 4: Commit**

```bash
git add .claude/commands/write.md
git commit -m "feat: write.md 新增 --no-review 标志支持快速粗稿模式"
```

---

## Task 4：write.md --auto 新增 --force 子参数（Feature E1）

**Files:**
- Modify: `.claude/commands/write.md`

**Step 1: 更新 argument-hint**

```
[章节号] [--words=3000] [--auto] [--auto --force] [--no-review]
```

**Step 2: 更新第三步参数解析**

新增：
```markdown
- **强制连续**：检查是否同时包含 `--auto` 和 `--force`（自动模式下章与章之间不暂停确认）
```

**Step 3: 更新第八步自动模式循环**

将原来的"询问用户：继续写第 N+1 章？"逻辑改为：

```markdown
3. 若 已完成章节数 < 大纲章节总数：
   - 告知用户："第 N 章完成。正在准备第 N+1 章……"
   - 显示下一章大纲概要（若存在）
   - **若包含 `--force`**：直接继续，不等待确认
   - **若不包含 `--force`**：询问用户："继续写第 N+1 章？（回复'继续'开始，回复'停止'结束自动写作）"
```

**Step 4: 人工验证**

确认：
- `--auto` 单独使用时仍会每章暂停（保持原有行为，符合 README 描述）
- `--auto --force` 才会跳过暂停

**Step 5: Commit**

```bash
git add .claude/commands/write.md
git commit -m "feat: write.md --auto 模式新增 --force 参数支持真正无干预连续写作"
```

---

## Task 5：write.md 添加上下文截断策略（Feature F5）

**Files:**
- Modify: `.claude/commands/write.md`

**Step 1: 在第五步末尾添加截断说明**

在"将所有读取到的内容整理为'写作素材包'"之前插入：

```markdown
**上下文长度控制：**
- `outline/overview.md` 超过 3000 字时，仅提取以下部分传给 Agent：
  - 基本信息表格（完整）
  - 人物档案（完整）
  - 故事结构（完整）
  - 章节总览（仅传递目标章节前后各5行，即 N-5 到 N+5 章的总览行）
  - 写作备注（完整）
- `style-rules.md` 超过 4000 字时，仅传递第一节至第八节，跳过第九节（示例段落）直接传递第十节（AI味检测清单）和第十一节（禁止总清单）
- 前序正文截取规则：第 N-1 章取末尾 **800字**，第 N-2 章取末尾 **300字**（比原来精确，原始说法"末尾500字/200字"改为更宽裕的值以保留完整段落）
```

**Step 2: 同步更新 Writer Agent 提示中的引用说明**

将 Writer Agent 提示中的：
```
- 第[N-1]章正文末尾500字：[内容或"这是第一章"]
- 第[N-2]章正文末尾200字：[内容或"不存在"]
```
改为：
```
- 第[N-1]章正文末尾800字（保留完整段落）：[内容或"这是第一章"]
- 第[N-2]章正文末尾300字（保留完整段落）：[内容或"不存在"]
```

**Step 3: 人工验证**

检查第五步和第六步的 Writer/Reviewer 提示，确认截断规则被前后一致地引用。

**Step 4: Commit**

```bash
git add .claude/commands/write.md
git commit -m "feat: write.md 添加上下文截断策略，防止大型项目超出 Agent 上下文窗口"
```

---

## Task 6：outline.md 新增"补充单章大纲"模式（Feature F4）

**Files:**
- Modify: `.claude/commands/outline.md`

**Step 1: 更新 argument-hint 和模式判断**

`argument-hint` 改为：
```
[--auto] [--chapter=N] [简短构思描述]
```

在第二步"模式判断"末尾新增第三个分支：

```markdown
- 包含 `--chapter=N` → 进入【单章补充模式】
```

**Step 2: 在交互模式和自动模式之间插入单章补充模式**

```markdown
## 【单章补充模式】（--chapter=N）

从 `$ARGUMENTS` 中提取 N 值（章节编号）。

### 前置检查
- 若 `outline/overview.md` 不存在：告知用户"需要先有整体大纲才能补充单章大纲，请先运行 `/outline` 或 `/outline --auto`"，停止执行。
- 若 `outline/chapter-NN.md` 已存在：展示现有内容，询问用户"是重写还是在此基础上修改？"

### 读取上下文
读取以下文件（若存在）：
1. `outline/overview.md`（了解整体故事结构和章节在全书中的位置）
2. `outline/chapter-(N-1).md`（前章大纲，了解衔接需求）
3. `outline/chapter-(N+1).md`（后章大纲，了解本章需要为其铺垫什么）

### 单章信息收集
基于整体大纲的"章节总览"表格中第 N 章的记录，询问用户：
1. "第 N 章核心事件的具体细节是什么？（基于大纲概要可以追问）"
2. "有什么需要特别注意的衔接点或伏笔？"

### 生成并保存
按【大纲标准格式】中的 `outline/chapter-XX.md` 格式生成单章大纲，保存至 `outline/chapter-NN.md`。

完成后告知用户："第 N 章大纲已保存至 `outline/chapter-NN.md`"。
```

**Step 3: 人工验证**

确认：
- `--chapter=N` 与 `--auto` 不会同时生效（`--chapter` 优先级更高，若同时出现进入单章补充模式）
- 单章模式下不会覆盖整体大纲 `overview.md`

**Step 4: Commit**

```bash
git add .claude/commands/outline.md
git commit -m "feat: outline.md 新增 --chapter=N 单章大纲补充模式"
```

---

## Task 7：新建 /status 命令（Feature F2）

**Files:**
- Create: `.claude/commands/status.md`

**Step 1: 创建文件**

文件内容：

```markdown
---
description: 显示当前小说项目的写作进度总览。
argument-hint: （无参数）
allowed-tools: Read, Bash
---

你是写作进度分析助手。快速扫描项目文件并呈现结构化进度报告。

## 第一步：收集数据

依次运行：

- 书名：!`head -1 outline/overview.md 2>/dev/null | sed 's/^# //' || echo "未找到大纲"`
- 预计章节总数：!`grep -m1 '预计章节' outline/overview.md 2>/dev/null | grep -o '[0-9]*' | head -1 || echo "未知"`
- 大纲章节文件数：!`ls outline/chapter-*.md 2>/dev/null | wc -l | tr -d ' '`
- 已写章节文件数：!`ls chapters/chapter-*.md 2>/dev/null | wc -l | tr -d ' '`
- 已写章节列表：!`ls chapters/chapter-*.md 2>/dev/null | sort || echo "无"`
- 风格规范：!`test -f style-rules.md && echo "EXISTS" || echo "MISSING"`

## 第二步：计算总字数

对每个已写章节文件，使用 Read 读取内容后统计字符数（中文字符数约等于字数）。

汇总全部章节字数，计算总计。

> 若章节较多（>20章），可使用 Bash：
> `wc -m chapters/chapter-*.md 2>/dev/null | tail -1`

## 第三步：生成进度报告

以结构化格式输出：

---

**《[书名]》写作进度**

| 项目 | 状态 |
|------|------|
| 整体大纲 | ✓ 已完成 / ✗ 未创建 |
| 风格规范 | ✓ 已创建 / ✗ 未创建 |
| 章节大纲 | X 章已规划 / 预计 Y 章 |
| 已写章节 | X 章 / Y 章 |
| 总字数 | 约 X 字 |
| 进度 | X% |

**已完成章节：**
第1章 ✓ | 第2章 ✓ | 第3章 ✓ | ...

**待写章节（有大纲）：**
第X章 | 第X+1章 | ...

**缺少大纲的章节（无法直接写作）：**
[若有，列出；若无，显示"无"]

---

若项目文件均不存在，提示用户："当前目录不是一个 claude-novel-writeFlow 项目，请先运行 `/outline` 创建大纲。"
```

**Step 2: 人工验证**

检查逻辑：
- 各 bash 命令是否会在空目录下安全执行（有 `2>/dev/null` 保护）
- 进度百分比计算是否合理（已写章节数 / 大纲章节总数）
- 缺少大纲章节的检测：比较 `chapters/chapter-*.md` 和 `outline/chapter-*.md` 的差集

**Step 3: Commit**

```bash
git add .claude/commands/status.md
git commit -m "feat: 新增 /status 命令，显示写作进度总览"
```

---

## Task 8：新建 /export 命令（Feature F1）

**Files:**
- Create: `.claude/commands/export.md`

**Step 1: 创建文件**

```markdown
---
description: 将所有已写章节合并导出为单一完整稿件文件。
argument-hint: [--output=文件名] [--format=markdown|txt]
allowed-tools: Read, Write, Bash
---

你是稿件导出助手。将已写章节按顺序合并为完整稿件。

用户输入参数：$ARGUMENTS

## 第一步：解析参数

- `--output=文件名`：导出文件名（不含扩展名），默认为 `manuscript`
- `--format=markdown|txt`：导出格式，默认 `markdown`

## 第二步：环境检查

- 已写章节文件：!`ls chapters/chapter-*.md 2>/dev/null | sort || echo "无"`
- 大纲书名：!`head -1 outline/overview.md 2>/dev/null | sed 's/^# //' || echo "未命名小说"`

若无任何章节文件，告知用户"尚无已写章节，请先运行 `/write` 命令写作章节。"，停止执行。

## 第三步：合并章节

按章节编号升序，依次读取所有 `chapters/chapter-*.md` 文件内容。

组装为完整稿件，格式如下：

**Markdown 格式（--format=markdown，默认）：**
```
# [书名]

---

[第1章完整内容]

---

[第2章完整内容]

---

...
```

**TXT 格式（--format=txt）：**
将 Markdown 标题符号（`#`）和分隔线（`---`）去除，保留纯文字。

## 第四步：保存导出文件

- Markdown：保存至 `[output].md`（根目录下）
- TXT：保存至 `[output].txt`（根目录下）

保存成功后告知用户：
"✓ 稿件已导出至 `[output].[ext]`，共 X 章，约 XXXX 字。"

## 错误处理

- 若某章文件读取失败：跳过该章，在输出末尾注明"注意：第X章读取失败，已跳过"
- 若输出文件已存在：询问用户是否覆盖
```

**Step 2: 人工验证**

确认：
- 按章节编号升序排列（`sort` 保证正确顺序）
- Markdown 和 TXT 两种格式的章节间分隔方式正确
- 默认输出文件名为 `manuscript.md`

**Step 3: Commit**

```bash
git add .claude/commands/export.md
git commit -m "feat: 新增 /export 命令，支持将所有章节合并导出为完整稿件"
```

---

## Task 9：更新 README.md

**Files:**
- Modify: `README.md`

**Step 1: 更新命令列表**

在"命令详解"部分新增 `/status` 和 `/export` 的说明小节。

**Step 2: 更新"推荐工作流"**

```
第一步：/style          → 设定写作风格
第二步：/outline        → 创建小说大纲
第三步：/write --auto   → 自动写作全书
第四步：/status         → 查看进度总览
第五步：/export         → 导出完整稿件
```

**Step 3: 修正 AI味检测清单数量**

确认 README 中"8条机械感排查项"的描述现在与 style.md 实际条目数量一致。

**Step 4: 新增参数说明**

在 `/write` 参数表格中补充：
| `--no-review` | 跳过审核智能体，直接保存初稿（粗稿阶段使用） |
| `--auto --force` | 无干预连续写作，章与章之间不暂停确认 |

在 `/outline` 参数说明中补充：
```
/outline --chapter=N
```
> 仅生成或重写第 N 章的细纲，不影响整体大纲。

**Step 5: Commit**

```bash
git add README.md
git commit -m "docs: 更新 README，补充新命令和新参数说明"
```

---

## 执行顺序建议

Tasks 1-5 之间无依赖，可以并行。Task 6、7、8 独立，也可并行。Task 9 依赖所有前置任务完成。

```
Tasks 1, 2, 3, 4, 5, 6 → 可并行
Tasks 7, 8             → 可并行
Task 9                 → 最后执行
```

总计：9个任务，约20-30分钟完成所有修改。
