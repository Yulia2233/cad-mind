# bench_cli 使用指南

[`bench_cli`](https://github.com/Yulia2233/bench_cli) 是 STEP 重建实验的命令行 benchmark 工具，包含单件 oneshot/agent 评测和 `assembly/bench_assembly` 多 solid 评测。源码已发布在独立仓库 `main`；`step-8B-sft/bench_cli/` 仍保留实验工作区副本。以下命令均从独立仓库根目录运行。

## 环境与安全

需要 Node.js 22+、可运行的 CAD Code 安装和用于评分的 Python 环境。单件 agent 评测通过 `cad-code` CLI 创建真实会话；最终候选在独立、无法读取目标 STEP/IR 的环境中回放，再由可信评分器读取目标。

API profile 用交互输入密钥，或通过 stdin 从 secrets manager 传入。不要把 API key 写进命令行参数、prompt、数据集或 run manifest。基准目标属于 evaluation-only 数据，不得复制进训练数据；每个 run 固定所用 dataset revision、prompt 和输入 hash。

## 单件评测

从独立 `bench_cli` 仓库根目录运行（私有数据与 CAD Code 路径通过 `BENCH_DATA_DIR`、`BENCH_CAD_CODE_DIR` 配置）：

```bash
node ./bench_cli.mjs profile set lab \
  --base-url https://provider.example/v1 --model model-id
secret-manager-command | node ./bench_cli.mjs profile set lab \
  --base-url https://provider.example/v1 --model model-id --api-key-stdin
node ./bench_cli.mjs profile test lab
node ./bench_cli.mjs dataset list
node ./bench_cli.mjs run oneshot \
  --dataset oneshot270-v1 --profile lab --parallel 1
node ./bench_cli.mjs run agent \
  --dataset agent30-cadcode-v3 --profile lab --parallel 1
```

`oneshot` 发送一条 prompt，不使用工具或会话历史；`agent` 通过 CAD Code 使用真实 Agent 工具和多轮会话。并发表示独立 case 数，不是 best-of-N。调试时可用 `--limit N` 或 `--cases id,id` 控制范围。

Built-in datasets 是只读快照。修改前创建新 revision：

```bash
node ./bench_cli.mjs dataset clone oneshot270-v1 oneshot270-local
node ./bench_cli.mjs dataset add oneshot270-local \
  --id extra-001 --split proxy --target /path/part.step \
  --ir /path/part.ir.jsonl --prompt-file /path/prompt.txt
node ./bench_cli.mjs dataset revisions oneshot270-local
```

运行记录在 `.runtime/runs/<run-id>/`。Agent 自报分数不作为验收结果；独立 replay 和评分才决定 case 状态。rIoU、严格 B-rep IoU、运行失败、超时和缺少候选代码应分别看待。

## 多 solid assembly benchmark

assembly 入口是 `assembly/bench_assembly`，不替代单件命令。`run` 收集已完成 assembly attempt 并保存输入快照，不执行候选代码；`score` 随后调用独立 assembly evaluator：

```bash
./assembly/bench_assembly inspect --project PROJECT_DIR
./assembly/bench_assembly run --project PROJECT_DIR \
  --out .runtime/assembly-runs
./assembly/bench_assembly score --project PROJECT_DIR \
  --attempt-dir PROJECT_DIR/assembly/attempt-001 \
  --out .runtime/assembly-score
./assembly/bench_assembly history list
node --test assembly/test/*.test.mjs
```

项目需要 `project.json`、`assembly.ir.json` 和实际生成的 attempt 输入/输出。评测会校验路径、STEPIR v6.3 版本、SHA256 和 symlink。`assembly-verdict-v1` 分开报告几何、occurrence 覆盖、约束状态和 placement fallback；Agent 完成消息本身不代表通过。

装配优化无需改 benchmark 入口：`run` 仍委托 `cad-code-multi assemble`。新版控制器给 Agent 提供有界索引与分页，预先生成覆盖所有 occurrence 的 ground + fixed spanning tree，Agent 仅对少量有证据的候选改用运动副。完整 STEPIR/IR 仍由控制器和独立评测核对；单 solid 的命令、提示词和轨迹格式不变。

新版 benchmark 的 run 快照会核对并复制 manifest 声明的 context index/pages，在 `source/input/` 下保留相应 hash；没有 context index 的旧 attempt 仍兼容。请同时归档原 assembly attempt 目录、完整交付代码树、报告与 raw session export；`score` 仍读取原 attempt，run 快照不是独立回放包，不能只保留它就删除原目录。详见 [独立仓库的装配说明](https://github.com/Yulia2233/bench_cli/blob/main/assembly/README.md#large-assembly-context-and-static-defaults)。

## 历史与清理

```bash
node ./bench_cli.mjs history list
node ./bench_cli.mjs history compare RUN_A RUN_B
node ./bench_cli.mjs clean workspace --run RUN_ID
node ./bench_cli.mjs clean traces --run RUN_ID --yes
node --test test/*.test.mjs
```

清理操作会预览或要求确认。workspace 清理保留会话轨迹和分数；trace 清理移除选中轨迹记录但保留评分。删除前确认 run ID 与预览范围。单件和 assembly 的历史结果、SDK 版本及数据集 revision 应分开比较。
