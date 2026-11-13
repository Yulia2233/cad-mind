# CAD Mind

CAD Mind 是 CAD Agent 与 STEP 逆向工程项目的导航入口。各仓库按职责拆分：运行 Agent、生成 STEPIR、提供建模技能，以及管理实验与数据。

## 仓库导航

| 仓库 | 主要职责 | 从这里开始 |
|---|---|---|
| [cad-code](https://github.com/Yulia2233/cad-code) | OpenCode CAD Agent，提供 CLI、Web、建模会话、多 solid 分流与装配运行 | [使用指南](docs/cad-code.md) · [仓库 README](https://github.com/Yulia2233/cad-code/blob/main/README.md) |
| [step-tools](https://github.com/Yulia2233/step-tools) | STEPIR v6.3 生成、按需查询、评分与多 solid STEP 拆分/去重 | [使用指南](docs/step-tools.md) · [仓库 README](https://github.com/Yulia2233/step-tools/blob/main/README.md) |
| [cad-reverse-engineer](https://github.com/Yulia2233/cad-reverse-engineer) | SimpleCADAPI 2.2.7 Agent Skill：单件逆向技能，以及仅在装配时启用的 assembly 扩展 | [使用指南](docs/cad-reverse-engineer.md) · [仓库 README](https://github.com/Yulia2233/cad-reverse-engineer/blob/main/README.md) |
| [bench_cli](https://github.com/Yulia2233/bench_cli) | 单件 oneshot/agent benchmark 与独立多 solid assembly benchmark | [使用指南](docs/bench_cli.md) · [GitHub 仓库](https://github.com/Yulia2233/bench_cli) |
| [data-review](https://github.com/Yulia2233/data-review) | 本地轨迹人工审核网页：左侧完整 CAD Code/Qwen 轨迹，右侧 Target/Rebuild 等测图，✅/❌ 标注自动跳转 | [使用指南](docs/data-review.md) · [GitHub 仓库](https://github.com/Yulia2233/data-review) |
| `step-8B-sft`（本地仓库） | SFT/Agent 实验、数据集、提示词和批量轨迹队列；当前没有 GitHub remote。现有 `bench_cli/` 源码仍在此工作区 | [使用指南](docs/step-8B-sft.md) · GitHub 仓库待创建 |

## 按任务选择

- 用现有模型交互、重建单件或处理多 solid 项目：从 `cad-code` 开始。
- 查看或生成 STEPIR，验证 STEP 特征记录，拆分多 solid STEP：用 `step-tools`。
- 安装或修改 Agent 的 SimpleCADAPI 建模规范：用 `cad-reverse-engineer`。单件继续使用原 `simplecadapi` skill；只有装配会话加载 assembly 扩展。
- 运行单件或装配 benchmark：用独立仓库 `bench_cli`；实验工作区也保留兼容副本。
- 人工审核 CAD Code 轨迹、对照 Target/Rebuild 并保留标注：用 `data-review`，运行在本机 localhost。
- 管理训练数据、实验和批量 CAD Code 轨迹任务：用 `step-8B-sft`。

## 多 solid 流程

```text
step-tools 预处理 STEP
    -> 为每个 unique part 生成 STEPIR v6.3
    -> cad-code 按原单件流程重建每个 unique part
    -> 显式启动 assembly 对话并组合 notebook
    -> 独立回放与评估
    -> step-8B-sft bench / 队列 / 数据打包
```

单件 STEP 仍走 `cad-code` 原有会话和 skill；assembly 扩展只描述多文件组合。不要把不同 STEPIR 版本的面号混用，也不要把评测目标放入训练数据。

大装配先由脚本生成覆盖所有 occurrence 的静态 ground + fixed spanning tree；Agent 通过有界索引和分页按需读局部证据，只对有限候选推断运动副。悬空或不确定的 occurrence 保留源位置并继续 fixed。运行与归档细节见 [CAD Code 指南](docs/cad-code.md#大装配的上下文和默认约束) 和 [benchmark 指南](docs/bench_cli.md)。

## 当前功能分支

以下分支包含本轮多 solid 功能；PR 尚未创建：

- [cad-code `codex/multisolid-v63`](https://github.com/Yulia2233/cad-code/tree/codex/multisolid-v63)
- [step-tools `codex/multisolid-preprocessor`](https://github.com/Yulia2233/step-tools/tree/codex/multisolid-preprocessor)
- [cad-reverse-engineer `codex/assembly-extension`](https://github.com/Yulia2233/cad-reverse-engineer/tree/codex/assembly-extension)

`step-8B-sft` 的多 solid 批量队列目前只在本地根仓库；benchmark 源码已发布到独立 `bench_cli` 仓库。等 `step-8B-sft` 创建 GitHub remote 后，再更新上表和指南中的链接。
