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

用 Bash 创建目录：

```bash
mkdir -p context
```

依次用 Write 工具创建以下四个文件（若文件已存在则跳过）：

**`context/characters.md`**：
```
# 人物档案

> 记录正文中已确立的人物细节（外貌、口头禅、伤情、能力变化等）。
> 格式：`- 条目描述 \`chXX\``
> 规则：仅记录正文散文中具体写出的内容，大纲中的计划内容不在此列。
```

**`context/world.md`**：
```
# 世界设定

> 记录正文中已出现的地名、规则、道具、势力、组织等设定细节。
> 格式：`- 条目描述 \`chXX\``
```

**`context/continuity.md`**：
```
# 关键事件

> 记录正文中发生的重大事件，供后续章节保持一致性。
> 格式：`- [chXX] 事件描述`
```

**`context/timeline.md`**：
```
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

若包含 `--rebuild`，清空四个文件内容，仅保留文件头部的说明注释行。

告知用户："正在分析已有章节，重建上下文档案……"

### 分析阶段

读取所有已写章节文件列表，按章节编号升序排序。

**逐章分析：** 对每个章节文件，先读取该章节正文内容，再读取当前四个 context 文件的最新内容，然后启动一个 **Context Extractor Agent**，提供以下提示：

```
你是上下文提取智能体。从以下章节中提取"已确立事实"，更新到 context 文件中。

【第N章正文】
[章节完整内容]

【当前 context 文件内容】
characters.md：[当前内容]
world.md：[当前内容]
continuity.md：[当前内容]
timeline.md：[当前内容]

【提取规则（严格遵守）】
1. 只提取正文散文中具体描写出来的细节，大纲式的计划内容不算已确立事实
2. 对照现有 context 内容，已有记录的条目直接跳过（去重）
3. 若本章对已有条目产生变化（伤情痊愈、状态改变、关系转变），在原条目末尾追加变化：
   格式：`- 右臂骨折 \`ch45\` → 痊愈 \`ch52\``
4. 不删除已有条目，只新增或追加变化
5. 保持现有文件的格式，不修改头部说明行

【输出格式】
对每个需要变更的文件，输出完整的新文件内容（含原内容+本次新增/修改）。
若某文件无需变更，写"[文件名]：无变更"。

characters.md 新内容：[完整文件内容或"无变更"]
world.md 新内容：[完整文件内容或"无变更"]
continuity.md 新内容：[完整文件内容或"无变更"]
timeline.md 新内容：[完整文件内容或"无变更"]
```

收到 Extractor Agent 输出后，对有新内容的文件用 Write 工具覆盖写入。

### 完成报告

所有章节分析完毕后，告知用户：
"✓ context/ 重建完成，共分析 X 章，更新记录如下：
- characters.md：X 条记录
- world.md：X 条记录
- continuity.md：X 个关键事件
- timeline.md：X 个时间节点"

---

## 错误处理

- 若无任何已写章节且无 `--init` 参数：告知"无已写章节可分析，已创建空 context 文件。开始写作后将自动更新。"
- 若某章文件读取失败：跳过该章，完成后在报告中注明
- 若 Context Extractor Agent 调用失败：跳过该章，在报告中注明，建议稍后重新运行 `/context`
