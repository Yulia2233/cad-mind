# step-8B-sft 使用指南

`step-8B-sft` 是本地实验工程，集中管理 STEP 逆向训练/Agent 实验、数据集、benchmark、提示词和批量轨迹生成脚本。当前 Git 根目录没有 GitHub remote；它不是 `cad-code`、`step-tools` 或 `cad-reverse-engineer` 的替代品。

## 主要目录

| 目录 | 用途 |
|---|---|
| `datasets/` | 分版本、分任务的数据集与打包产物 |
| `bench_cli/` | 单件与多 solid benchmark；详见 [bench_cli 使用指南](bench_cli.md) |
| `stepir_geom_gen/` | 批量轨迹合成队列；历史队列按批次保留，新多 solid 队列在 `cadcode_synth_multisolid/` |
| `stepir_geom/` | STEPIR 研究工作树；多 solid 预处理已同步到 `Yulia2233/step-tools` 的功能分支 |
| `more_solid-data/` | 多实体开发验证 STEP 和 manifest；来源/用途依 manifest 记录 |
| `work/analysis/` | 调研、评测报告和开发验证证据 |
| `envs/` | 项目使用的本地 Python/工具环境配置 |

先读项目根目录的 `README.md`、`PITFALLS.md`、`NEXT_PLAN.md`，再执行数据或训练相关任务。评测数据不能进入训练集；正式打包应使用仓库脚本进行 STEP hash 与来源核对。密钥只通过环境变量、stdin 或权限受限的 secrets 文件传递。

## 单件 benchmark

从仓库根目录运行：

```bash
node ./bench_cli/bench_cli.mjs dataset list
node ./bench_cli/bench_cli.mjs run oneshot --dataset oneshot270-v1 --profile lab --parallel 1
node ./bench_cli/bench_cli.mjs run agent --dataset agent30-cadcode-v3 --profile lab --parallel 1
node --test bench_cli/test/*.test.mjs
```

`oneshot` 是无工具单轮请求；`agent` 通过 cad-code CLI 使用真实会话和工具。评分在模型运行结束后独立执行。built-in dataset 是只读快照，改数据先 clone 出新 revision。服务 profile 的 API key 由交互或 stdin 配置，不写进 prompt、run manifest 或日志。

## Multi-solid benchmark 与批量队列

benchmark 不运行候选代码的 agent 端入口和独立 score 分开：

```bash
./bench_cli/assembly/bench_assembly inspect --project PROJECT_DIR
./bench_cli/assembly/bench_assembly run --project PROJECT_DIR --out bench_cli/.runtime/assembly-runs
./bench_cli/assembly/bench_assembly score --project PROJECT_DIR \
  --attempt-dir PROJECT_DIR/assembly/attempt-001 \
  --out bench_cli/.runtime/assembly-score
node --test bench_cli/assembly/test/*.test.mjs
```

批量队列每个 STEP 建立一个项目，unique part 队列最多并行 40 个；项目之间串行，不叠乘并发。启动真实运行前先在 `cad-code` 配置模型、SimpleCADAPI 2.2.7、STEPIR v6.3 脚本和单件 prompt：

```bash
cd stepir_geom_gen/cadcode_synth_multisolid
python3 prepare.py /path/to/assembly.step --batch /path/to/batch \
  --cad-code-multi /path/to/cad-code/cad-code-multi
python3 run_projects.py --batch /path/to/batch --parallel 4 \
  --prompt-file /path/to/prompt_v63.md
python3 summarize.py --batch /path/to/batch
python3 recover.py --batch /path/to/batch --parallel 4
```

装配是显式阶段：调用 `run_projects.py --assemble` 前先确认 child 结果已就绪。长时间服务器队列放在 tmux，停止/恢复只操作该项目记录的会话。

同步与打包：

```bash
python3 sync_from_server.py --server aliyun --remote-batch /server/batch \
  --local-batch /path/to/local-batch
python3 pack_dataset.py --batch /path/to/local-batch \
  --dest datasets/multisolid-traj --dry-run
python3 -m unittest discover \
  -s stepir_geom_gen/cadcode_synth_multisolid/tests -p 'test_*.py' -v
```

打包器只接收独立 `assembly-verdict-v1` 通过、来源合规并有完整原始会话导出的项目；默认数据目录是 `datasets/multisolid-traj`，不会修改 `formal-data-traj` 或旧单件数据集。`cleanup_done.py` 默认只检查；只有确认缓存归属、归档完成和项目无活动任务后，才使用其 `--apply`。

## Git 和数据边界

当前本地根仓库包含大量数据、历史实验和未跟踪文件。不要用 `git add .`，不要在新建 GitHub 仓库前把整个工作区推上去。应按审阅范围只提交源代码、轻量清单和文档；数据集、STEP、轨迹和密钥按各自来源/访问规则单独管理。待仓库 remote 创建后，再将这个指南中的“本地工程”链接更新为实际 URL。
