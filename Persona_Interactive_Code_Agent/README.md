# Persona Interactive Code Agent

一个轻量的本地代码 Agent 原型，项目内部品牌名为 `pico`。它把大模型包在一个受控运行时里，让模型只能通过白名单工具读取、搜索、执行命令、写文件或打补丁，并把每次运行的会话、记忆、checkpoint、trace 和 benchmark 指标落盘，方便恢复和复盘。

## 项目亮点

- 受控工具执行：模型输出必须解析成 `<tool>` 或 `<final>`，工具调用会先经过参数校验、路径边界检查、重复调用拦截和审批策略。
- 多后端模型适配：支持 Ollama、OpenAI-compatible `/responses` 接口，以及 Anthropic-compatible `/messages` 接口。
- 会话恢复：`.pico/sessions` 保存可恢复会话，checkpoint 会记录当前目标、下一步、关键文件 freshness 和 runtime identity。
- 分层记忆：短期工作记忆、相关记忆召回、文件摘要缓存，以及可持久化的 topic memory。
- 上下文预算控制：按 prefix、memory、relevant memory、history 和 current request 分段渲染 prompt，超预算时按固定策略压缩。
- 审计工件：每次 `ask()` 会写入 `.pico/runs/<run_id>/task_state.json`、`trace.jsonl` 和 `report.json`。
- Benchmark harness：内置 deterministic fixture 任务和 metrics/ablation 脚本，用来评估工具边界、恢复、记忆和上下文压缩效果。

## 目录结构

```text
Persona_Interactive_Code_Agent/
  cli.py              # CLI 参数解析和 Pico 装配
  runtime.py          # Agent 控制循环、工具执行护栏、session/checkpoint
  models.py           # Ollama / OpenAI-compatible / Anthropic-compatible 后端
  tools.py            # 工具白名单、参数校验和执行逻辑
  workspace.py        # 工作区快照与 prompt prefix 基线信息
  context_manager.py  # prompt 分段组装和预算压缩
  memory.py           # 工作记忆、文件摘要、durable topic memory
  run_store.py        # 单次运行工件落盘
  evaluator.py        # 固定 benchmark harness
  metrics.py          # metrics 聚合和实验工具
benchmarks/
  coding_tasks.json   # 固定 benchmark 任务定义
scripts/
  collect_resume_metrics.py
  run_large_scale_experiments.py
  run_provider_experiments.py
tests/
  ...                 # pytest 测试与 benchmark fixtures
```

## 快速开始

可以直接从源码目录运行：

```powershell
python -m Persona_Interactive_Code_Agent --help
```

启动交互式 agent：

```powershell
python -m Persona_Interactive_Code_Agent --cwd . --provider ollama --model qwen3.5:4b
```

执行一次 one-shot 请求：

```powershell
python -m Persona_Interactive_Code_Agent --cwd . --provider ollama "列出项目结构并总结入口文件"
```

## 模型配置

### Ollama

先启动本地 Ollama 服务，并确保模型已拉取：

```powershell
ollama serve
ollama pull qwen3.5:4b
python -m Persona_Interactive_Code_Agent --provider ollama --model qwen3.5:4b
```

常用参数：

- `--host`：Ollama 地址，默认 `http://127.0.0.1:11434`
- `--temperature`：默认 `0.2`
- `--top-p`：默认 `0.9`

### OpenAI-compatible

```powershell
$env:OPENAI_API_KEY="your_api_key"
python -m Persona_Interactive_Code_Agent --provider openai --model gpt-5.4 --base-url https://www.right.codes/codex/v1
```

相关环境变量：

- `OPENAI_API_KEY`
- `OPENAI_MODEL`
- `OPENAI_API_BASE`

### Anthropic-compatible

```powershell
$env:ANTHROPIC_API_KEY="your_api_key"
python -m Persona_Interactive_Code_Agent --provider anthropic --model claude-sonnet-4-6 --base-url https://www.right.codes/claude/v1
```

相关环境变量：

- `ANTHROPIC_API_KEY`
- `ANTHROPIC_MODEL`
- `ANTHROPIC_API_BASE`
- `RIGHT_CODES_API_KEY`

## CLI 常用命令

交互模式内支持：

- `/help`：显示命令帮助
- `/memory`：查看当前工作记忆
- `/session`：查看 session 文件路径
- `/reset`：清空当前 session 历史和记忆
- `/exit` 或 `/quit`：退出

恢复最近会话：

```powershell
python -m Persona_Interactive_Code_Agent --resume latest
```

指定审批策略：

```powershell
python -m Persona_Interactive_Code_Agent --approval ask
python -m Persona_Interactive_Code_Agent --approval auto
python -m Persona_Interactive_Code_Agent --approval never
```

`run_shell`、`write_file`、`patch_file` 属于 risky tool，会受审批策略控制。

## 安全边界

工具层会做几类关键限制：

- 所有文件路径都解析到 workspace root 下，防止 `../` 或符号链接逃逸。
- 写文件、打补丁、运行 shell 命令默认需要审批。
- shell 执行只传入 allowlist 环境变量，减少敏感信息暴露。
- trace/report 会对配置的 secret 环境变量进行脱敏。
- `patch_file` 必须精确匹配且只匹配一次，避免模糊修改。
- 连续重复的相同工具调用会被拒绝，防止模型卡循环。

可以通过 `--secret-env-name` 或 `PICO_SECRET_ENV_NAMES` 增加需要脱敏的环境变量名。

## 运行测试

```powershell
python -m pytest -q
```


## Benchmark 和实验

固定 benchmark 任务定义在 `benchmarks/coding_tasks.json`，覆盖文档编辑、文本 patch、工具边界、恢复和 durable memory 合同。

可以从 Python 中运行 deterministic benchmark：

```powershell
python -c "from Persona_Interactive_Code_Agent.evaluator import run_harness_regression_v2; run_harness_regression_v2()"
```

收集 resume/记忆/上下文/安全指标：

```powershell
python scripts\collect_resume_metrics.py `
  --benchmark-artifact artifacts\harness-regression-v2.json `
  --runs-root artifacts\provider-workspaces `
  --output-json artifacts\resume-metrics.json `
  --output-markdown artifacts\resume-metrics.md
```
