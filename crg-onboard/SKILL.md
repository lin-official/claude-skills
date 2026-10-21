---
name: crg-onboard
description: 把 code-review-graph 知识图谱接入当前仓库——安装、MCP、hooks、CLAUDE.md、建图、嵌入、验证;幂等可重跑,兼任升级维护
disable-model-invocation: true
---

# CRG 接入

把 code-review-graph(CRG)接入当前仓库。**幂等**:每步先检测、缺什么补什么;重跑 = 检查 + 补缺 + 升级,同一流程兼任首次接入与日常维护。

全局变量:`ROOT` = 仓库根(步骤 0),`PY` = Miniconda python(步骤 1)。命令均在 git-bash 下运行。

## 步骤 0|前提

```bash
cd <目标仓库> && ROOT=$(git rev-parse --show-toplevel)
```

非 git 仓库(常见:导出快照)→ 终止并告知用户;典型处置 = 就地 `git init`(本地分析场景,无远程也无妨),由用户拍板。

完成判据:`$ROOT` 非空。

## 步骤 1|Python 侧(Miniconda base)

```bash
PY=$(command -v python)                      # 确认指向 Miniconda
$PY -m pip show code-review-graph            # 缺 → $PY -m pip install code-review-graph
$PY -c "import sentence_transformers"        # 失败 → $PY -m pip install sentence-transformers
$PY -c "import igraph"                       # 失败 → $PY -m pip install igraph
code-review-graph -v
```

已装 CRG 时报告版本;changelog 见上游 <https://github.com/tirth8205/code-review-graph>(仅参考,安装走 PyPI)。用户要升级 → `$PY -m pip install -U code-review-graph`。

完成判据:`code-review-graph -v` 正常输出,三个 import 全过。

## 步骤 2|.mcp.json

1. `PYWIN=$(cygpath -w "$PY")`、`ROOTWIN=$(cygpath -w "$ROOT")`(JSON 里反斜杠双写)。
2. `.mcp.json` 缺 `mcpServers["code-review-graph"]` → 写入下条;已有其他 server → 合并追加,保留原条目:

```json
{
  "mcpServers": {
    "code-review-graph": {
      "command": "<PYWIN>",
      "args": ["-m", "code_review_graph", "serve"],
      "cwd": "<ROOTWIN>",
      "env": {"PYTHONUTF8": "1", "CRG_EMBEDDING_MODEL": "BAAI/bge-small-zh-v1.5"},
      "type": "stdio"
    }
  }
}
```

3. `.gitignore` 补 `.mcp.json`(机器本地文件:绝对路径每台机器不同,重跑本 skill 再生)。`.mcp.json` 已被 git 跟踪时 gitignore 不生效——报告即可。

完成判据:`$PY -c "import json;json.load(open('.mcp.json'))"` 通过且含 crg 条目;`git check-ignore .mcp.json` 命中。

## 步骤 3|hooks 两件套

`.claude/settings.json` 缺则创建;已有 hooks → 合并追加,保留现有条目:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "cat >/dev/null || true; command -v code-review-graph >/dev/null 2>&1 || exit 0; git rev-parse --git-dir >/dev/null 2>&1 && code-review-graph update --skip-flows --repo \"$(git rev-parse --show-toplevel 2>/dev/null)\" || true",
            "timeout": 30
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "cat >/dev/null || true; command -v code-review-graph >/dev/null 2>&1 || exit 0; git rev-parse --git-dir >/dev/null 2>&1 && code-review-graph status --repo \"$(git rev-parse --show-toplevel 2>/dev/null)\" || echo 'Not a git repo, skipping'",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

完成判据:JSON 校验通过,两条 command 字符串均在文件中。

## 步骤 4|图数据目录

`.gitignore` 补 `.code-review-graph/`。

完成判据:`git check-ignore .code-review-graph/` 命中。

## 步骤 5|CLAUDE.md 标准通用段

模板 = 本 skill 目录 `templates/claudemd-section.md`(首尾带 `crg-onboard:v1` 标记)。`CLAUDE.md` 含标记 → 整段替换(升级);无标记、无文件 → 追加/创建。

完成判据:CLAUDE.md 含成对标记且中间为模板内容。

## 步骤 6|建图

已有图(`status` 有节点数)→ `code-review-graph update --repo "$ROOT"`;否则 `code-review-graph build --repo "$ROOT"`。

完成判据:`code-review-graph status --repo "$ROOT"` 节点/边 > 0。

## 步骤 7|嵌入

```bash
code-review-graph embed --provider local --model BAAI/bge-small-zh-v1.5 --repo "$ROOT"
```

模型统一 `BAAI/bge-small-zh-v1.5`;换模型的代价见排障①。

完成判据:embed 报告写入向量数 > 0。

## 步骤 8|验证

1. CLI 冒烟(**必须带 env**,原因见排障①):

```bash
CRG_EMBEDDING_MODEL=BAAI/bge-small-zh-v1.5 PYTHONUTF8=1 \
  code-review-graph search "<取自该仓库 README/模块名的领域词>" --repo "$ROOT" --limit 5
```

2. 告知用户:重启会话(或 /mcp 重连)后 MCP 生效;hook 在下次 Edit/Write 与新会话生效。

完成判据:冒烟搜索返回 ≥ 1 条。

---

## 新机器自举

1. 部署 skills 集合仓(含本 skill):`git clone https://github.com/lin-official/claude-skills.git ~/.claude/skills`(目录本体即仓库,已存在则 `git pull`)。
2. 装 Miniconda,然后 `pip install code-review-graph sentence-transformers igraph`。
3. 首次 embed 需从 HuggingFace 下载模型(见排障⑤)。
4. 每个仓库重跑 `/crg-onboard`——`.mcp.json` 按新机路径重生成,图全量重建,`.gitignore`/hooks/CLAUDE.md 已提交的部分原样生效。

## 排障

① **语义搜索静默返 0**:CRG 按 `provider:model` 身份键过滤向量,查询端与嵌入端模型不一致时不报错、直接 0 结果。MCP 端由 `.mcp.json` 的 `CRG_EMBEDDING_MODEL` 对齐;CLI 查询必须带同款 env。换模型 = 重跑 embed + 两端 env 同改。
② **增量更新不嵌新节点**:build/update 默认不嵌入;代码大改后手动跑步骤 7(按 text_hash 增量,重复跑便宜)。
③ **hook 里的 `update --skip-flows` 跳过 flows/社区检测**:需要时手动 `code-review-graph postprocess --repo "$ROOT"`。
④ **CLI 中文输出乱码**:命令前缀 `PYTHONUTF8=1`。
⑤ **embed 首次下载模型慢/超时**(HuggingFace,~100MB):前缀 `HF_ENDPOINT=https://hf-mirror.com`。
⑥ 云嵌入 provider 会外发源码派生文本——本项目一律 `--provider local`。
