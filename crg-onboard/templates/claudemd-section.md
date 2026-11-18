<!-- crg-onboard:v2 开始 -->
## MCP Tools: code-review-graph

**本项目有知识图谱。先用 code-review-graph MCP 工具收窄范围,再读源码。** 图比扫文件便宜,且给出文件搜索给不了的结构信息(调用方、依赖方、测试覆盖)。

### 先用图工具的场景

- **探索代码**:`semantic_search_nodes_tool` 或 `query_graph_tool` 代替 Grep
- **评估影响**:`get_impact_radius_tool` 代替手工追 import
- **代码评审**:`detect_changes_tool` + `get_review_context_tool` 代替通读全文件
- **查关系**:`query_graph_tool` 的 callers_of / callees_of / imports_of / tests_for
- **架构问题**:`get_architecture_overview_tool` + `list_communities_tool`

### 以源码为准

- 用图收窄范围,然后读源码;改动从不只依据图输出。
- 非平凡改动前,读实现与相关测试。
- 触及行为、数据库、迁移、重试、回退、兼容代码时,核对确切源码。
- 图与源码冲突时,源码赢——图可能过期或未建模该关系。
- 图结果为空可能是"未索引"或"静态不可见",不等于"不存在"。

### 工具速查

| 工具 | 场景 |
| ------ | ---------- |
| `detect_changes_tool` | 评审变更——风险评分分析 |
| `get_review_context_tool` | 评审要源码切片——省 token |
| `get_impact_radius_tool` | 变更波及面 |
| `get_affected_flows_tool` | 受影响的执行路径 |
| `query_graph_tool` | 追调用方/被调/导入/测试 |
| `semantic_search_nodes_tool` | 按名字/关键词找函数类 |
| `get_architecture_overview_tool` | 高层结构 |

### 保鲜与自动注入(hooks)

- 图自动保鲜:Edit/Write 后增量更新;commit/checkout/merge/rebase 后 git hooks 同步。
- **自动注入(steering)**:每次 Grep/Glob/Read 前自动附加 `[code-review-graph]` 上下文(文件内符号→行号、调用关系,约 100–300 tokens)。据此直接定点 Read(offset/limit),不整读文件;Grep 前先看注入结果,常可跳过盲搜。
- 会话开场先 `get_minimal_context_tool(task="<任务>")` 拿摘要与建议工具。

### 分级阅读协议(省 token 标准流程)

1. **L0** `get_minimal_context_tool` — 开场摘要与建议工具。
2. **L1** `semantic_search_nodes_tool(include_source=true, expand_neighbors=true)` — 一次拿到代码及其调用方/被调;多数问题到此为止,不开文件。
3. **L2** `query_graph_tool` / `get_flow_tool` — 追调用、导入、测试、执行路径;默认 `detail_level="minimal"`。
4. **L3** 定点 `Read(offset, limit)` — 用 L1/L2/注入给出的行号读切片;大文件先 `query_graph_tool pattern=file_summary` 看结构再选段,禁无界整读。

预算:单任务约 5 次图调用、800 tokens 图输出。图只收窄范围,不替代源码(见"以源码为准")。

标准入口:`/explore-codebase` 探索、`/review-changes` 评审、`/debug-issue` 调试、`/refactor-safely` 重构。
<!-- crg-onboard:v2 结束 -->
