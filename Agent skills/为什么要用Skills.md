---
date: 2026-06-02
aliases:
  - my first note about Agent skill
tags:
  - Skill
---
# 什么是Skills
- Skill是一种把“提示词+操作流程+参考资料+脚本”封装成可复用能力包的方法，让通用Agent不必每次都重新学习任务规则，只在需要时加载需要的专业知识。

# 如何打包一项Skill
## 1. 创建SKILL.md
## 2. 每项SKILL.md开头都需要添加名称和描述
- **名称**辅助模型进行参考
- **描述**辅助模型理解何时使用此特定技能
``` Markdown
---  
name: Your-skill-name  
description: Your skill description  
---

```
## 3. 文件打包
以`Your-skill-name`命名文件夹，把SKILL.md放入其中，再在`Your-skill-name`文件夹中新建子文件夹`references`，把写着复杂规则的md放入`references`文件夹中

# Skills的适用场景：
| Category  类别 | Examples  示例                          |
| ------------ | ------------------------------------- |
| **领域专业知识**   | 品牌规范和模板，法律审查流程，数据分析方法                 |
| **可重复的工作流程** | 每周营销活动回顾，客户电话准备流程，季度业务回顾              |
| **新功能**      | 创建演示文稿、生成 Excel 表格或 PDF 报告、构建 MCP 服务器 |

# 如果没有Skills
- 每次都需要重复输入命令和需求
- 每次都需要将参考文献和支撑文件捆绑在一起
- 手动确保工作流程输出始终保持一致

# Skills的关键特征：
- 便携性：可以在不同兼容技能的代理之间重用相同的技能
- 可组合性：技能可以组合来构建复杂的流程
	- 比如：
		- 公司品牌创办（字体、颜色、标志）
		- 创建幻灯片
		- 市场营销模式
		- 分析营销数据

# 技能如何工作
| 层                                                                            | 加载时   |
| ---------------------------------------------------------------------------- | ----- |
| **Metadata** (YAML frontmatter: name, description)  <br>元数据（YAML 前置内容：名称，描述） | 始终加载  |
| **Instructions** (main SKILL.md content)  <br>说明（主 SKILL.md 内容）              | 触发时加载 |
| **Resources** (reference files, scripts)  <br>资源（参考文件、脚本）                    | 按需加载  |
## Skill包示例：
- SKILL.md 是必需的
- 可选目录：参考文献、脚本和资源
``` Markdown
analyzing-marketing-campaign/
├── SKILL.md
└── references/
    └── budget_reallocation_rules.md

```

``` Markdown
pdf/
├── SKILL.md
├── forms.md
├── reference.md
└── scripts/
    ├── check_fillable_fields.py
    ├── convert_pdf_to_images.py
    ├── extract_form_field_info.py
    └── fill_pdf_form_with_annotations.py
```

``` Markdown
designing-newsletters/
├── SKILL.md
├── references/
│   └── style-guide.md
└── assets/
    ├── header.png
    ├── icons/
    └── templates/
        ├── newsletter.html
        └── layout.docx
```
