# data-review 使用指南

[`data-review`](https://github.com/Yulia2233/data-review) 是 CAD Code 轨迹的本地人工审核网页。它把完整轨迹放在左侧，把 Target 和 Rebuild 的等测图放在右侧；标注只提供 ✅ 和 ❌，保存后自动跳到下一条未标注记录。

仓库发布网页、服务端、目录构建脚本、渲染脚本和明确标注的合成演示样例。真实轨迹、STEP、图片缓存、人工标注和 API key 都留在本地实验目录，不进入公开仓库。

## 启动

从 `data-review` 仓库目录执行：

```bash
git clone https://github.com/Yulia2233/data-review.git
cd data-review
python3 server.py --root . --catalog examples/catalog.json \
  --data-dir .runtime/demo --images examples/images --port 8778
```

此命令显示仓库内的合成演示，标注只写 `.runtime/demo/annotations/`。演示轨迹不是实际 Agent 运行，示例 box 的 STEP 和等测图由 SimpleCADAPI 2.2.7 生成。

连接已有的 `step-8B-sft` 批注目录时，使用：

```bash
python3 server.py \
  --root /path/to/step-8B-sft \
  --data-dir /path/to/step-8B-sft/sites/v63-trajectory-labeler \
  --catalog /path/to/step-8B-sft/sites/v63-trajectory-labeler/active-catalog.json \
  --images /path/to/step-8B-sft/sites/v63-trajectory-labeler/images \
  --annotations /path/to/step-8B-sft/sites/v63-trajectory-labeler/annotations \
  --port 8778
```

打开 <http://127.0.0.1:8778/>。服务端只监听本机，网页不会执行轨迹中的命令或 notebook。

参数含义：

- `--root`：轨迹清单中相对路径的根目录。
- `--data-dir`：catalog、图片状态和其他本地网页数据目录。
- `--catalog`：要显示的 catalog JSON。可以冻结一个批次或全部记录。
- `--images`：`target.png`、`rebuild.png` 和 `render.json` 所在目录。
- `--annotations`：保存 `marks.json` 和追加写入的 `history.jsonl` 的目录。

### 可选渲染

网页服务只读取已有图片。需要从 STEP 重新生成等测图时，在原实验工作区使用 SimpleCADAPI 2.2.7：

```bash
export DATA_REVIEW_ROOT=/path/to/step-8B-sft
export DATA_REVIEW_DATA_DIR=/path/to/step-8B-sft/sites/v63-trajectory-labeler
export DATA_REVIEW_CATALOG="$DATA_REVIEW_DATA_DIR/active-catalog.json"
/path/to/python-with-simplecadapi-2.2.7 render_pairs.py --scope all --workers 3
```

从独立仓库运行渲染脚本时，以上环境变量指定原实验数据和输出目录；公开仓库不包含 STEP 和图片缓存。渲染使用相同相机和包围盒时，两侧图片可直接对照；无 rebuild 的记录会明确显示缺失。查看已渲染的图片只需 Python 标准库，渲染步骤才需要 CAD SDK。

## 标注与数据边界

`GET /api/catalog` 返回当前目录的记录和已有标注；`GET /api/trajectory` 返回完整轨迹；`GET /api/image` 返回等测图；`GET/POST /api/marks` 保存标注。服务端校验本机 Host、Origin 和页面 token，标注文件采用原子写入，并追加保存修改历史。

轨迹中的工具参数、工具返回、错误、媒体和原始 JSON 均保留。Qwen 数据可以通过 `qwen_adapter.py` 显示 `reasoning_content`；事后补写解释会标注来源，不伪装成原始隐藏思考。

审核结果不会自动进入训练数据。确认收录后，回到 `step-8B-sft` 使用对应的打包脚本，按 STEP hash、独立 verdict、轨迹和来源规则生成数据集。

## 安全与发布边界

这是本地审核工具，不是公网部署服务，也不是 GitHub Pages 应用。不要把以下内容提交到 `data-review`：

- `annotations/`、`images/`、`.qa/`；
- Target/Rebuild STEP、model.py、trajectory JSON 和包含模型请求的日志；
- API key、provider 配置和带密钥的运行环境。

公开仓库中的示例命令使用占位路径。实际数据位置通过命令行参数传入，便于在不同实验目录之间切换。
