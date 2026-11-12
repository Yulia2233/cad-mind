# cad-reverse-engineer 使用指南

此仓库维护 SimpleCADAPI 2.2.7 Agent Skill 的源码和构建产物。单件逆向入口是原 `skill/`；新增的 `simplecadapi-assembly` 是 opt-in 扩展，只由装配会话加载。

## 单件 skill

```bash
git clone https://github.com/Yulia2233/cad-reverse-engineer.git
cd cad-reverse-engineer
```

源码位于 `skill/`，编译后的 Codex、Claude Code、OpenCode 等版本位于 `builds/`。安装前确认目标 Agent 对应的目录名为 `simplecadapi`，再复制适配平台的 build：

```bash
cp -R builds/simplecadapi-codex ~/.codex/skills/simplecadapi
```

STEP 逆向以 STEPIR v6.3 为输入，产出 SimpleCADAPI 2.2.7 的 marimo notebook。常用辅助脚本：

```bash
python <skill-root>/scripts/stepir.py target.step --out target.ir.jsonl
python <skill-root>/scripts/stepeval.py model.py target.step --out eval/attempt-1
```

## 编译与校验

修改单件 `skill/` 后，使用共享 skill builder 编译并检查：

```bash
python tools/skillbuild.py --check
python tools/skillbuild.py --target opencode
python tools/skillbuild.py
```

编译器要求 SimpleCADAPI skill/compiler 依赖可用。正式安装前核对 `skills/manifest.json` 和构建结果。这个 builder 面向原单件 skill。

## Assembly 扩展

多 solid 项目的次序是：先用普通单件流程重建每个 unique part，再在**单独的装配对话**中加载 `extensions/simplecadapi-assembly/`。不要把装配说明加入单件 system prompt，也不要修改或替换原 `skill/`。

扩展要求 SimpleCADAPI 2.2.7、STEPIR v6.3、多份已选 child notebook 和 `step-assembly-ir-v1`。它保留重复 occurrence，并对悬空或证据不足的零件采用源 STEP placement 作为静态 fallback；这不代表推断出了机械约束。

```bash
python tools/assembly_skillbuild.py --check
python tools/assembly_skillbuild.py --out /tmp/simplecadapi-assembly-opencode
python -m unittest discover -s tests -p 'test_assembly_extension.py' -v
```

默认构建目录 `builds/simplecadapi-assembly-opencode` 必须不存在；builder 不覆盖已有目录。扩展 build 与单件 build 分开安装，cad-code 的 assembly runtime 默认从 `tools/assembly-skill` 加载它，也可通过 `CAD_CODE_ASSEMBLY_SKILL` 指定安装路径。

## 安装依赖与安全边界

建议在独立虚拟环境使用：

```bash
python -m pip install "simplecadapi[editor,verify]==2.2.7" manifold3d
```

技能源码和 build 中不应放 API key。评测目标、装配 IR 和 notebook 输入由调用方管理；装配回放只读已核验源代码和 SDK，目标 STEP 不应暴露给回放进程。此扩展是面向装配的增量能力，不改变原轨迹格式和单件工作流。
