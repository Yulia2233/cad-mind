# cad-code 使用指南

`cad-code` 是基于 OpenCode 的本地 CAD Agent。CLI 和 Web 共用会话、工作区和模型配置。单件工作流使用 SimpleCADAPI 2.2.7 skill；多 solid 工作流在此基础上增加项目管理和独立装配会话。

## 安装与配置

需要 Node.js 22+、OpenCode 1.18.27、Python 3.12/3.13 和对应的 SimpleCADAPI 环境。macOS 单件沙箱依赖系统 `sandbox-exec`；Linux 执行需要 bubblewrap，并且隔离不可用时会拒绝运行。

```bash
git clone https://github.com/Yulia2233/cad-code.git
cd cad-code
npm install -g opencode-ai@1.18.27
npm ci
npm run setup -- --python python3.12
```

配置模型 API 时通过交互输入密钥，或从标准输入传入，避免把密钥写进 shell 历史：

```bash
./cad-code config --base-url https://your-provider.example/v1
secret-manager-command | ./cad-code config --base-url https://your-provider.example/v1 --api-key-stdin
./cad-code config --show
```

配置保存在 `.runtime/` 的私密文件中，不要提交密钥文件。

## 单件逆向

```bash
./cad-code --web
./cad-code --file ./part.step run "Reconstruct this part and export a STEP file."
./cad-code run "Create a 20 mm tall cylinder with a 10 mm radius."
./cad-code --continue
```

新工作区位于 `workspace/<session-id>/`。输入会复制到会话 `input/`，不会移动原文件。单件 prompt、STEPIR v6.3 格式和独立 `stepeval.py` 验收由现有 skill 管理。

## 多 solid 项目

多 solid 自动分流目前在 `codex/multisolid-v63` 分支。单 solid 输入仍委托原 CLI；多 solid 输入生成独立项目，先为 unique part 创建普通单件会话，再由用户显式启动装配会话。

```bash
git switch codex/multisolid-v63
export CAD_CODE_PYTHON="$PWD/.runtime/python/bin/python"
export CAD_CODE_MULTISOLID_PART_PROMPT=/path/to/prompt_v63.md
./cad-code --file ./assembly.step run "Reconstruct this STEP."
```

也可以显式运行项目命令：

```bash
./cad-code-multi inspect --file assembly.step
./cad-code-multi prepare --file assembly.step --out workspace/projects/demo
./cad-code-multi run-parts --project workspace/projects/demo --parallel 4 --prompt-file /path/to/prompt_v63.md
./cad-code-multi status --project workspace/projects/demo --json
./cad-code-multi assemble --project workspace/projects/demo --wait --timeout 600
./cad-code-multi export --project workspace/projects/demo --out exports/demo
./cad-code-multi --web
```

并发限制为 1–40 个 unique part worker。零件逆向仍使用原单件 prompt；assembly skill 只在装配会话启用。Linux 独立装配回放还需设置 `CAD_CODE_BWRAP` 指向可执行的 bubblewrap，并使用 SimpleCADAPI 2.2.7 Python 环境。装配 evaluator 只挂载哈希核对过的代码树、SDK 和独立 scratch；目标 STEP/IR 不进入回放环境。

项目状态、选定的 child attempt、assembly attempt 和 `assembly-verdict-v1` 保存在项目目录中。对话完成不代表评估通过，应检查 `geometry_pass`、occurrence coverage 和 `mate_status`。

### 大装配的上下文和默认约束

装配优化版由控制器核对完整 assembly IR、child notebook 和 STEPIR v6.3 的 hash，但 Agent 首先只读 `input/assembly-context-index.json`，再按需读取 `input/assembly-index/` 中的分页。不会把上百份 STEPIR 一次放进初始提示词。索引会记录已提供和省略的候选关系数量；未检查的候选保留为待确认假设。

控制器在开始装配对话前生成 `out/product.py`：每个 unique part 定义只复用一次，每个 occurrence 保留 STEP 中的位置；固定一个参考件，并用 fixed spanning tree 约束其余 occurrence。Agent 只对有限候选检查局部证据，有几何和功能依据才将某条 fixed 边换成运动副；不得留下无约束 occurrence。悬空或不确定的零件继续 fixed。这样的静态默认保证位置与覆盖完整，不代表已恢复原设计的机械运动。

分析或归档时应保留原 attempt 的完整 `input/`、交付的 child-source 代码树、装配报告、独立 verdict 和原始会话导出。单件逆向继续使用原 prompt、skill、运行路径和轨迹 schema。

## 数据、导出和测试

```bash
./cad-code list
./cad-code export ses_... --output trajectory.json
./cad-code delete workspace ses_...
npm test
```

会话导出保留原始 `cad-code-session-v2` 消息与工具媒体。项目导出位于 `multi/export.mjs`；服务器端长任务使用独立 tmux 会话，并只停止本项目拥有的进程。
