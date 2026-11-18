# claude-skills

用户级 Claude Code skills 集合。本仓库即 `~/.claude/skills` 目录本体,新机器一条命令部署:

```bash
git clone https://github.com/lin-official/claude-skills.git ~/.claude/skills
```

## 内容

- **crg-onboard** — code-review-graph 跨仓库接入标准流程:幂等八步(安装/MCP/hooks 含 steering 读侧注入与 git hooks/CLAUDE.md v2 分级阅读协议/建图/嵌入/验证),兼任首次接入与升级维护;内含「新机器自举」与「排障」(含 hook 静默失效)两节
- **code-documentation** — 代码文档生成
