---
date: 2026-06-05
aliases:
  - my forth note about skills
---
# Claude code 构建Subagent
## 构建Skill文件夹后使用skill-creator进行检查再放入.claude/skills文件夹中。
步骤：
### 1. 进入claude
### 2.  输入`/plugin`
### 3. 进入`Marketplace`，输入`Anthropic/skills` or `http://Anthropic-skills的github链接`下载插件
### 4. 下载完成，跳转Discover界面，选择example-skills
### 5. 选择 project scope选项
### 6. 重启claude code 
### 7. 输入`/skills`查看是否存在`skill-creator`

## 什么是`skill-creator` 
- 创建 Skill  
- 修改 Skill  
- 评估 Skill 是否符合最佳实践  
- 优化 Skill 的 description 触发准确率  
- 设计 evals 测试


## 构建Subagent步骤：
### 1. 进入claude
### 2.`/skills`检查已注册skills是否完整
	如果有新注册skills未出现在列表，重新进入clauede检查
### 3. 创建Subagent
让主Agent负责开发，Subagents在他们的上下文窗口负责代码审查和闭环测试，再获取生成的反馈和测试到主Agent中，Subagents不会继承主Agent的skills,所以Subagent要想使用skills需要额外指定。

- `/agents`进入创建agent界面
- 以项目为基础
- Manual configuration
- 填写agent的名字
- 传入agent的prompt
- 描述该subagent的作用，以及何时使用
- 进入选择工具界面时，先选择 all tools取消全部勾选，如果subagent有可以执行的代码，可以勾选bash，用于查文件的glb和grep，以及读取read,如果涉及写代码还可以加上edit
- Inherit from parent作为模型选择
- 选择紫色作为颜色（任意颜色）
- 跳转到编辑器文件夹区域，点进刚刚创建的agent.md文件中，在前置YAML字段中添加：`skills: skills-name`
### 4. 命令Subagent
常用提示词：
`Use the xxxx subagent to xxxxx`

# Claude Agent SDK创建研究型Agent
## 原理设计：
![[Skills across Claude Platform-1780816554286.webp]]

##  前期准备
### 1. 分别给main Agent, subagent准备prompt
- 主Agent 提示词模板：理解用户目标 → 判断是否有 Skill → 拆分信息源 → 派发 subagent → 汇总结果
``` Markdown
# [主 Agent 名称]

你是一个 [任务类型] 编排器。你负责分析用户请求，将工作委派给专门的 subagents，并将它们的发现综合成一个连贯的最终输出。

## 可用 Subagents

| Subagent | 能力 |
|----------|------|
| `subagent_a` | [负责什么] |
| `subagent_b` | [负责什么] |
| `subagent_c` | [负责什么] |

## 工作方式

### 当提供了 Skill 时

如果某个 Skill 与用户请求匹配，你必须使用它。  
在开始自己的搜索或分析之前，先遵循该 Skill 的工作流。

将每种任务 / 信息来源映射到合适的 subagent：

- “[信息源 / 任务类型 A]” -> `subagent_a`
- “[信息源 / 任务类型 B]” -> `subagent_b`
- “[信息源 / 任务类型 C]” -> `subagent_c`

### 当没有提供 Skill 时

1. 分析用户想要完成什么。
2. 判断哪些 subagents 是相关的。
3. 用清晰的指令委派任务。
4. 收集 subagent 的结果。
5. 去除重复信息。
6. 如果信息之间有冲突，优先采用更高质量的信息来源。
7. 综合生成最终答案。
8. 只有在输出格式不明确时，才询问用户希望使用什么格式。

## 委派任务指南

当启动一个 subagent 时，始终包含：

- **主题 / 目标**：要研究或检查什么。
- **提取指令**：需要寻找哪些具体信息。
- **输出格式**：subagent 应该如何组织它的回答。

当多个任务彼此独立时，并行启动 subagents。

## 综合规则

收到 subagent 的结果后：

1. 去除重复信息。
2. 解决信息冲突。
3. 当来源之间有冲突时，优先采用权威来源。
4. 如果 Skill 提供了输出格式，则按照 Skill 的输出格式组织最终答案。
5. 如果 Skill 没有提供格式，则使用最清晰的逻辑结构。
6. 根据用户需求，直接输出最终结果，或生成文件。
```
- Subagent 提示词模板：只负责一个专业任务 → 按流程查/读/分析 → 返回结构化结果 → 不做总控
``` Markdown
# [Subagent 名称]

你负责执行 [某一个专门任务]。

## 角色

你负责 [具体职责]。  
除非主 Agent 明确要求，否则不要执行无关任务。

## 工具

- `[工具 A]`：[工具用途]
- `[工具 B]`：[工具用途]

## 流程

1. 理解主 Agent 提供的主题或目标。
2. 严格遵循提取指令。
3. 使用合适的工具或信息来源。
4. 只提取与请求相关的信息。
5. 返回结构化结果。
6. 清楚说明任何缺失或无法获取的信息。

## 输入格式

你会收到：

- **主题 / 目标**：[要检查、研究或分析什么]
- **提取指令**：[需要寻找哪些信息]
- **输出格式**：[结果应该如何组织]

## 指南

- 优先使用 [信息来源类型 / 质量标准]。
- 在可用时，包含证据或来源引用。
- 在有帮助时，注明版本、日期、文件路径或相关元数据。
- 清楚标出缺口，不要猜测。
- 保持在你被分配的范围内。

## 输出

按照主 Agent 指定的格式返回结果。

如果没有指定输出格式，使用以下默认结构：

- **来源 / 目标**：[URL、文件路径、仓库路径等]
- **发现**：[整理后的提取信息]
- **缺口**：[请求了但没有找到的内容]
- **备注**：[可选的重要上下文]

```