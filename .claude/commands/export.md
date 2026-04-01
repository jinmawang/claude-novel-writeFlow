---
description: 将所有已写章节合并导出为单一完整稿件文件。
argument-hint: [--output=文件名] [--format=markdown|txt]
allowed-tools: Read, Write, Bash
---

你是稿件导出助手。将所有已写章节按顺序合并为完整稿件文件。

用户输入参数：$ARGUMENTS

## 第一步：解析参数

从 `$ARGUMENTS` 中提取：
- `--output=文件名`：导出文件名（不含扩展名），默认为 `manuscript`
- `--format=markdown|txt`：导出格式，默认 `markdown`

## 第二步：环境检查

运行以下检查：

- 已写章节文件：!`ls chapters/chapter-*.md 2>/dev/null | sort || echo "无"`
- 大纲书名：!`head -1 outline/overview.md 2>/dev/null | sed 's/^# //' || echo "未命名小说"`
- 导出文件是否已存在（markdown）：!`test -f manuscript.md && echo "EXISTS" || echo "OK"`
- 导出文件是否已存在（txt）：!`test -f manuscript.txt && echo "EXISTS" || echo "OK"`

**若无任何章节文件：** 告知用户"尚无已写章节，请先运行 `/write` 命令写作章节。"，停止执行。

**若导出文件已存在：** 询问用户是否覆盖（`[output].[ext]` 已存在，确认覆盖吗？）。

## 第三步：读取并合并章节

使用 Read 工具按章节编号升序，依次读取所有 `chapters/chapter-*.md` 文件内容（编号排序：01, 02, ..., 09, 10, 11, ...）。

若某章文件读取失败，跳过该章，在最终报告中注明。

## 第四步：组装稿件

**Markdown 格式（默认）：**

组装内容结构如下：
```
# [书名]

---

[第1章完整内容]

---

[第2章完整内容]

---

（以此类推，章节之间用 --- 分隔）
```

**TXT 格式（--format=txt）：**

将 Markdown 标记去除：
- 移除行首的 `#`、`##` 等标题符号（保留文字，转为纯文本标题行）
- 将 `---` 分隔线替换为空行
- 其余内容保持原样

## 第五步：保存导出文件

- Markdown：写入 `[output].md`（项目根目录）
- TXT：写入 `[output].txt`（项目根目录）

保存成功后告知用户：
"✓ 稿件已导出至 `[output].[ext]`，共 X 章，约 XXXX 字。"

若有章节读取失败，在结尾补充说明：
"注意：以下章节读取失败，已跳过：[章节列表]"

---

## 错误处理

- 若 chapters/ 目录不存在：告知用户尚无已写章节
- 若所有章节读取均失败：告知用户并建议检查文件权限
