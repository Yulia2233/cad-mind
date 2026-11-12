# bench_cli 使用指南

`bench_cli` 位于 `step-8B-sft/bench_cli/`，是 STEP 重建实验的命令行 benchmark 工具，不是独立 Git 仓库。它有两类入口：原有单件 oneshot/agent 评测，以及 `assembly/bench_assembly` 多 solid 评测。

## 环境与安全

需要 Node.js 22+、可运行的 CAD Code 安装和用于评分的 Python 环境。单件 agent 评测通过 `cad-code` CLI 创建真实会话；最终候选在独立、无法读取目标 STEP/IR 的环境中回放，再由可信评分器读取目标。

API profile 用交互输入密钥，或通过 stdin 从 secrets manager 传入。不要把 API key 写进命令行参数、prompt、数据集或 run manifest。基准目标属于 evaluation-only 数据，不得复制进训练数据；每个 run 固定所用 dataset revision、prompt 和输入 hash。

## 单件评测

从 `step-8B-sft` 根目录运行：

```bash
node ./bench_cli/bench_cli.mjs profile set lab \
  --base-url https://provider.example/v1 --model model-id
secret-manager-command | node ./bench_cli/bench_cli.mjs profile set lab \
  --base-url https://provider.example/v1 --model model-id --api-key-stdin
node ./bench_cli/bench_cli.mjs profile test lab
node ./bench_cli/bench_cli.mjs dataset list
node ./bench_cli/bench_cli.mjs run oneshot \
  --dataset oneshot270-v1 --profile lab --parallel 1
node ./bench_cli/bench_cli.mjs run agent \
  --dataset agent30-cadcode-v3 --profile lab --parallel 1
```

`oneshot` 发送一条 prompt，不使用工具或会话历史；`agent` 通过 CAD Code 使用真实 Agent 工具和多轮会话。并发表示独立 case 数，不是 best-of-N。调试时可用 `--limit N` 或 `--cases id,id` 控制范围。

Built-in datasets 是只读快照。修改前创建新 revision：

```bash
node ./bench_cli/bench_cli.mjs dataset clone oneshot270-v1 oneshot270-local
node ./bench_cli/bench_cli.mjs dataset add oneshot270-local \
  --id extra-001 --split proxy --target /path/part.step \
  --ir /path/part.ir.jsonl --prompt-file /path/prompt.txt
node ./bench_cli/bench_cli.mjs dataset revisions oneshot270-local
```

运行记录在 `bench_cli/.runtime/runs/<run-id>/`。Agent 自报分数不作为验收结果；独立 replay 和评分才决定 case 状态。rIoU、严格 B-rep IoU、运行失败、超时和缺少候选代码应分别看待。

## 多 solid assembly benchmark

assembly 入口是 `bench_cli/assembly/bench_assembly`，不替代单件命令。`run` 收集已完成 assembly attempt 并保存输入快照，不执行候选代码；`score` 随后调用独立 assembly evaluator：

```bash
./bench_cli/assembly/bench_assembly inspect --project PROJECT_DIR
./bench_cli/assembly/bench_assembly run --project PROJECT_DIR \
  --out bench_cli/.runtime/assembly-runs
./bench_cli/assembly/bench_assembly score --project PROJECT_DIR \
  --attempt-dir PROJECT_DIR/assembly/attempt-001 \
  --out bench_cli/.runtime/assembly-score
./bench_cli/assembly/bench_assembly history list
node --test bench_cli/assembly/test/*.test.mjs
```

项目需要 `project.json`、`assembly.ir.json` 和实际生成的 attempt 输入/输出。评测会校验路径、STEPIR v6.3 版本、SHA256 和 symlink。`assembly-verdict-v1` 分开报告几何、occurrence 覆盖、约束状态和 placement fallback；Agent 完成消息本身不代表通过。

## 历史与清理

```bash
node ./bench_cli/bench_cli.mjs history list
node ./bench_cli/bench_cli.mjs history compare RUN_A RUN_B
node ./bench_cli/bench_cli.mjs clean workspace --run RUN_ID
node ./bench_cli/bench_cli.mjs clean traces --run RUN_ID --yes
node --test bench_cli/test/*.test.mjs
```

清理操作会预览或要求确认。workspace 清理保留会话轨迹和分数；trace 清理移除选中轨迹记录但保留评分。删除前确认 run ID 与预览范围。单件和 assembly 的历史结果、SDK 版本及数据集 revision 应分开比较。
