Coding Agent 后续进展方案
本文档记录当前项目完成状态，以及后续建议的开发路线。目标是逐步把当前最小 coding agent 原型，扩展成结构清晰、可测试、可展示的学习项目。

当前阶段状态
当前项目已经完成 V0：最小可运行 coding agent。

已完成能力：

CLI agent 入口。
.env 自动加载。
OpenAI-compatible 配置。
responses / chat_completions 双模式。
DeepSeek 等第三方兼容服务可接入。
自定义 JSON agent loop。
原生 Tool Calling 可选模式。已完成。
--verbose 可查看 agent 执行过程。
--trace-file 可写入 JSONL 审计日志。已完成。
结构化 Action 解析。
dataclass 工具参数 Schema 和运行前校验。已完成。
JSON 错误恢复和连续错误上限。
轻量 Planner 分层。
项目级 Code Intelligence：入口、符号、import、依赖、测试映射和架构摘要。已完成。
SQLite session history 和可选历史恢复。已完成。
本地 chunk 检索工具 search_code_chunks。已完成。
只读工具：列文件、读文件、搜索文本。
安全编辑工具：write_file、apply_patch，包含敏感路径检查。
白名单命令工具：run_command，支持安全路径参数、pytest 失败解析和临时目录执行。已完成。
Review Mode 只读代码审查。已完成。
基础测试覆盖。
当前阶段目标已经完成。后续重点不是一次性做复杂功能，而是按阶段增强工程质量和 agent 能力。

Phase 1：可观测性（已完成）
目标：让开发者清楚看到 agent 每一步在做什么。

建议新增模块：

src/coding_agent/tracing.py
建议新增 CLI 参数：

--verbose
需要实现：

记录每一轮模型输入。
记录模型原始输出。
记录工具调用。
记录工具结果。
记录最终回答。
示例输出：

[step 1] model requested tool: list_files
[step 1] tool result: ok
[step 2] model returned final
验收标准：

默认模式保持简洁输出。已完成。
--verbose 模式展示 agent 执行过程。已完成。
测试覆盖 verbose 开启和关闭两种情况。已完成。
Phase 2：结构化 Action（已完成）
目标：不要继续用裸 dict 表示模型动作。

建议新增模块：

src/coding_agent/schema.py
建议定义：

@dataclass
class FinalAction:
    final: str


@dataclass
class ToolAction:
    tool: str
    args: dict
需要实现：

parse_action(raw: str)。
校验模型输出是否是 JSON 对象。
校验是否只包含一种动作。
校验 tool 是否是字符串。
校验 args 是否是字典。
验收标准：

agent.py 不再直接处理裸 JSON 字典。已完成。
非法动作会返回明确错误。已完成。
测试覆盖 final、tool、非法 JSON、非法字段类型。已完成。
Phase 3：错误恢复能力（已完成）
目标：模型输出不规范时，agent 不直接失败，而是尝试恢复。

需要实现：

JSON 解析失败时，把错误反馈给模型重试。
模型输出 Markdown 代码块时，尝试提取内部 JSON。
工具不存在时，让模型重新选择工具。
参数错误时，让模型修正参数。
连续错误超过限制后停止。
建议新增配置：

--max-errors 2
验收标准：

模型输出非标准 JSON 时可以恢复。已完成。
连续错误超过上限时停止并说明原因。已完成。
测试覆盖 JSON 解析失败、工具不存在、参数错误。已完成。
Phase 4A：Memory / Executor / Prompts 分层（已完成）
目标：先做结构重构，不引入复杂 planner。

已新增模块：

src/coding_agent/memory.py
src/coding_agent/executor.py
src/coding_agent/prompts.py
src/coding_agent/tools/factory.py
职责划分：

memory.py：保存单次任务的用户输入和 observation。
executor.py：执行 ToolAction 并记录 trace。
prompts.py：集中管理 agent system prompt。
tools/factory.py：集中创建默认工具集合。
验收标准：

agent.py 不再直接拼接 transcript 字符串。已完成。
agent.py 不再直接执行工具注册逻辑。已完成。
main.py 不再手动逐个注册工具。已完成。
测试覆盖 memory、executor、factory。已完成。
Phase 4B：Planner 分层（已完成）
目标：从“边想边做”升级为“先计划，再执行”。

已新增模块：

src/coding_agent/planner.py
职责划分：

planner.py：生成轻量本地计划。
memory.py：保存计划和 observation。
agent.py：负责调度 planner、executor 和 memory。
推荐流程：

用户任务
  -> 生成计划
  -> 执行第 1 步
  -> 观察结果
  -> 执行第 2 步
  -> 最终总结
验收标准：

agent 能生成简短计划。已完成。
--verbose 能输出计划。已完成。
计划会写入 memory 并传给模型。已完成。
Phase 5：更可靠的文件编辑（已完成）
目标：让 agent 更安全地修改代码。

当前已有：

apply_patch
建议增强：

patch 前检查修改路径。
禁止修改 .env。
禁止修改 .git。
patch 应用前展示影响文件。
patch 应用失败时返回详细错误。
修改后自动返回 git diff。
支持非 git 项目的简单写文件能力。
验收标准：

修改敏感文件会被拒绝。已完成。
patch 应用失败时错误清楚。已完成。
patch 成功后能展示 diff。已完成。
测试覆盖成功 patch、失败 patch、敏感文件 patch。已完成。
Phase 6：代码索引（已完成）
目标：让 agent 具备基础项目理解能力，而不是每次盲目搜索。

已新增模块：

src/coding_agent/indexer.py
第一版不需要向量数据库，只做简单索引：

文件树。
文件大小。
文件类型。
Python 文件中的 class/function 名。
README、pyproject、package.json 等关键文件识别。
已新增工具：

summarize_workspace
验收标准：

能识别项目入口文件。已完成。
能统计主要文件类型。已完成。
能输出简短项目摘要。已完成。
测试覆盖 Python 项目和敏感文件跳过。已完成。
Phase 7：命令执行增强（已完成）
目标：让 agent 更像开发助手，但仍保持安全。

当前已有：

run_command
后续建议：

支持 python -m pytest tests/test_x.py。
支持 python -m ruff check src tests。
支持根据项目类型推荐测试命令。
对失败测试输出做摘要。
当前仍不建议开放任意 shell 命令。

验收标准：

仍然使用白名单机制。已完成。
不允许危险命令。已完成。
测试失败时能把关键信息返回给模型。已完成。
Phase 8：面试展示材料（已完成）
目标：让项目可以放进简历，并且能在面试中讲清楚。

已新增：

docs/architecture.md
docs/demo.md
docs/security.md
建议内容：

架构图。
agent loop 流程。
工具调用流程。
安全设计。
DeepSeek / OpenAI 兼容设计。
示例运行输出。
验收标准：

README 能快速说明项目价值。已完成。
docs 能解释核心架构。已完成。
面试时能围绕 agent loop、工具系统、安全边界展开讲解。已完成。
推荐执行顺序
当前路线中的核心阶段已经完成。后续可以按需要继续增强。

不建议下一阶段立刻做：

向量数据库。
多 agent 协作。
Web UI。
任意 shell 命令执行。
这些功能复杂度较高，容易让学习项目失控。当前阶段更重要的是把 agent loop、工具调用、安全边界和错误恢复做扎实。

下一阶段最小目标
建议下一阶段定义为 V1：

V1 = 可观测、结构化、可恢复的 agent loop
V1 完成标准：

支持 --verbose。已完成。
支持 --trace-file JSONL 审计日志。已完成。
有结构化 Action。已完成。
支持自定义 JSON 协议和原生 Tool Calling 双模式。已完成。
JSON 错误能恢复。已完成。
工具调用日志清晰。已完成。
测试覆盖正常路径和错误路径。
README 能解释 agent loop。
