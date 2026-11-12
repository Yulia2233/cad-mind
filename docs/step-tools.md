# step-tools 使用指南

`step-tools` 维护 STEPIR 生成与评估工具。当前功能分支 `codex/multisolid-preprocessor` 增加 SimpleCADAPI 2.2.7 多 solid 拆分、保守去重和装配 IR 生成；它不改变既有单件 STEPIR v6.3 parser。

## 环境

推荐 Python 3.12。单件解析依赖 SimpleCADAPI/OCP；评估和预处理按具体脚本还会用到 `numpy`、`scipy`、`manifold3d`、`tokenizers` 等依赖。真实 benchmark 数据不随仓库分发。

```bash
git clone https://github.com/Yulia2233/step-tools.git
cd step-tools
git switch codex/multisolid-preprocessor
```

## 单件 STEPIR v6.3

```bash
python cand/stepir.py part.step
python cand/stepir_ops.py part.step --augment --fit
python cand/stepir_query.py part.step face 12
python cand/stepir_query.py part.step evidence 12
python cand/stepir_query.py part.step axis
python cand/stepir_query.py part.step neighbours 12
```

单文件 bundle 适合只允许安装一个工具文件的 Agent 工作区：

```bash
python bundle/build_bundle.py --out stepir.py
python stepir.py part.step
python stepir.py part.step --base
python stepir.py query part.step face 12
```

STEPIR v6.3 与 v6.2.x 的面编号及记录不兼容。生成 IR 与按面查询必须使用同一版本脚本；切勿把旧 IR 的面号交给 v6.3 查询。

## 多 solid 预处理

此功能将每个 solid occurrence 输出为独立 STEP，验证回写几何，严格去重，并为每个 unique part 生成 v6.3 IR 和 `assembly.ir.json`。只有严格等价检查通过才合并；未通过的输入会写 `unsupported_input` 诊断，不会放宽阈值。

```bash
export MULTISOLID_PYTHON=/path/to/python-with-simplecadapi-2.2.7
export MULTISOLID_STEPIR_SCRIPT=/path/to/frozen-stepir-v6.3.py
"$MULTISOLID_PYTHON" multisolid/cli.py inspect assembly.step
"$MULTISOLID_PYTHON" multisolid/cli.py prepare assembly.step \
  --out work/multisolid/demo \
  --python "$MULTISOLID_PYTHON" \
  --stepir-script "$MULTISOLID_STEPIR_SCRIPT"
```

`--stepir-script` 必须是内容 hash 匹配的冻结 v6.3 bundle。输出包括 `split/instance-NNN.step`、`unique/part-NNN/target.step`、`target.ir.jsonl`、`ir_version.txt`、split/dedup manifests 和装配 IR。输入树和输出目录应保留在受控项目路径；不要把评测专用目标打包成训练数据。

## 评估和测试

```bash
python bench/eval_geom.py --workers 8
python tools/verify_implied.py --help
"$MULTISOLID_PYTHON" -m unittest discover -s multisolid -p 'test_*.py' -v
```

`bench/` 用于研究评分，要求对应的本地基准数据和 manifest。不要将不同 STEPIR 版本的结果直接混在一个对比中。多 solid 预处理测试使用合成 fixture，可独立于真实 benchmark 运行。
