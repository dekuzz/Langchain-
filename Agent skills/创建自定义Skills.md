---
date: 2026-06-04
aliases:
  - my third note about skills
tags:
  - skills
---
# Skills 结构
## 1. YAML前置内容

|     字段名     | 条件                            |
| :---------: | ----------------------------- |
|    name     | - 最多64个字符                     |
|             | - 只能包含小写字母、数字、连字符             |
|             | - 不能包含xml标签                   |
|             | - 不能包含保留词“anthropic”、“claude” |
| description | - 必须非空                        |
|             | - 最多1024个字符                   |
|             | - 不能包含xml标签                   |
|             | - 应该包含描述技能的作用以及何时使用它          |
|             | - 应包含特定关键词，帮助代理识别相关任务         |


## 2. 可选前置字段
| Field  领域              | Constraints  约束     | 示例                                                               |                                                                        |                                               |
| ---------------------- | ------------------- | ---------------------------------------------------------------- | ---------------------------------------------------------------------- | --------------------------------------------- |
| **license  许可证**       | 许可证名称或指向许可证文件的引用    | `license: Proprietary. LICENSE.txt has complete terms`           |                                                                        |                                               |
| **compatibility  兼容性** | 最大 500 字符；表示环境要求    | `compatibility: Designed for Claude Code (or similar products)`, | `compatibility: Requires git, docker, jq, and access to the internet`, | `compatibility: Requires Python 3.14+ and uv` |
| **metadata  元数据**      | 任意键值对（例如，作者、版本）     | `metadata:<br>  author: example-org<br>  version: "1.0"`         |                                                                        |                                               |
| **allowed-tools**      | 空格分隔的预先批准的工具列表（实验性） | `allowed-tools: Bash(git:*) Bash(jq:*) Read`                     |                                                                        |                                               |


## 3. 正文内容
常规来说没有格式限制，但有
### **推荐章节编写：**
- **逐步说明**  *Step-by-step instructions*
- **输入输出的示例** *Examples of inputs and outputs*
- **常见边缘情况** *Common edge cases*
### **实用指南**：
- 500行以内
- 显示基本内容链接高级内容
- `references/` 文件夹通常和 `SKILL.md` 放在同一个 Skill 根目录下。
- 清晰简洁，术语全文保持一致
- 文件路径使用正斜杠

### **自由发挥空间**：
- **高度自由**：只说方向，不规定具体做法
	- 优点：灵活
	- 缺点：结果不稳定
	比如：
``` Markdown
Create a useful report based on the user's data.	
```
- **中等自由度**：推荐流程或代码模式，允许适当变化
	- 优点：稳定且不死板
	- 缺点：任务要求严格时依旧不稳定
	比如：
``` Markdown
Use this pattern when analyzing feedback:

1. Group feedback by theme
2. Count frequency for each theme
3. Extract representative quotes
4. Summarize key findings

You may adapt the categories based on the data.
```

- **低自由度**：必须严格按指定脚本或固定步骤来做
	- 优点：稳定，可复现，少出错
	- 缺点：灵活性差，脚本或流程不适配容易卡住
比如：
``` Markdown
Use `scripts/generate_report.py`.

Required command:
python scripts/generate_report.py --input data.csv --output report.docx

Do not create the report manually.

```

#### **对比表：**
| 自由度   | 你给 Agent 的约束 | 适合任务        | 特点        |
| ----- | ------------ | ----------- | --------- |
| 高自由度  | 只给目标         | 写作、总结、创意分析  | 灵活但不稳定    |
| 中等自由度 | 给推荐流程 / 示例模式 | 分析、报告、代码生成  | 稳定和灵活之间平衡 |
| 低自由度  | 给固定脚本 / 固定步骤 | 校验、转换、打包、测试 | 稳定但不灵活    |
### **复杂工作流程**：
- 将复杂操作分解为清晰、顺序的步骤
- 如果工作流变得很大且包含许多步骤，考虑将它们推入单独的文件
### **可选目录**：
#### **`/assets`**：
- `/Templates`:文档模板，配置模板
- `/Images`:图片、图标、标志
- `/Data files`: 查找表、架构
#### **`/references`**:
- 附加详细规则
- 保持单个文件参考专注
- ps:(超过100行的参考文件，顶部要包含目录，以便代理了解完整范围)
#### `/scripts`**：

- 记录依赖关系
- 清晰文档
- 明确且有帮助的错误处理
- ps: 应该明确说明Claude 是应该执行脚本还是将其作为参考

## 4. 评估
==评估json文件要放到/eval文件夹中==
#### **单元测试**
#### **定义测试用例**
- 要测试的Skills
- 用于运行的测试提示query
- 要使用的输入文件
- 预期的成功样子
#### **示例测试用例：**

```json
{
  "skills": ["generating-practice-questions"],
  "queries": [
    "Generate practice questions from this lecture note and save it to output.md",
    "Generate practice questions from this lecture note and save it to output.tex",
    "Generate practice questions from this lecture note and save it to output.pdf"
  ],
  "files": ["test-files/notes.pdf", "test-files/notes.tex", "test-files/notes.pdf"],
  "expected_behavior": [
    "Successfully reads and extracts the input file. For pdf input, uses pdfplumber.",
    "Successfully extracts all the learning objectives.",
    "Generates the 4 types of questions.",
    "Follows the guidelines for each question.",
    "Uses the output structure and the correct output templates.",
    "The latex output successfully compiles.",
    "Saves the generated questions to a file named output."
  ]
}
```

#### **其他评估技巧**：
- 人类反馈
- 以后运行这套skills的模型的测试


## Claude中构建skills过程
