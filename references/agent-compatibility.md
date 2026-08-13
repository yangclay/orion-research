# Agent 兼容矩阵

Orion Research 方法论与工具调用分离。以下映射帮助你在不同 Agent 环境中执行本 skill。

## 核心工具映射

| 本 skill 需要的能力 | Hermes | Claude Code / Claude Agent Skills | OpenCode / Cursor | 通用替代方案 |
|---|---|---|---|---|
| 代码执行（编排搜索管道） | `execute_code` | bash 工具 / `code_execution` | tool_code / bash | Python 脚本 + `subprocess` |
| 搜索 | 内置 `web_search` | WebSearch / WebFetch | 内置搜索工具 | Tavily API / SerpAPI / Brave API |
| 抓取单页 | `scripts/web-fetch.sh` | WebFetch / curl | curl | `curl` / `requests` |
| 反爬页面 | `scripts/scrape-stealth.py` | Playwright MCP | 浏览器工具 | Playwright / Scrapling |
| 整站抓取 | `scripts/crawl_site.py` | crawl4ai | crawl4ai | `crawl4ai` |
| 语义重排 | Jina Rerank API | Jina Rerank API | Jina Rerank API | Jina / Cohere Rerank |
| 记忆/经验记录 | Hermes 记忆系统 | CLAUDE.md 更新 / 记忆文件 | AGENTS.md 更新 | 追加到记忆文件 |
| API key 管理 | `.env`（按环境读取） | 环境变量 | 环境变量 | 环境变量（不要硬编码） |

## 安装方式

### Anthropic Claude / Claude Code（Agent Skills 标准）

将仓库内容放入 skills 目录：

```bash
# Claude Code
mkdir -p ~/.claude/skills/orion-research
cp -r SKILL.md references/ scripts/ gotchas/ evals/ ~/.claude/skills/orion-research/

# Claude Desktop / Claude.ai（按官方 Agent Skills 导入方式）
```

### Hermes Agent

```bash
git clone https://github.com/yangclay/orion-research.git \
  ~/.hermes/profiles/<profile>/skills/orion-research
```

### OpenCode / Cursor 等

将 `SKILL.md` + `references/` 放入项目的 skill/agent 指令目录（OpenCode: `.opencode/skills/`，Cursor: `.cursor/rules/`），按各自格式调整 frontmatter 即可。

## 各环境注意事项

- **Hermes**：完整支持，含 SaC 模板、限流控制、中文生态专项
- **Claude Code**：`execute_code` 用 bash 替代；记忆记录用更新 CLAUDE.md 替代；其余方法论通用
- **OpenCode/Cursor**：建议先跑 ReAct 模式（轻量），复杂调研再启用 CodeAct（需要可用的搜索 API key）
- **通用**：API key 一律从环境变量读取，禁止硬编码进任何文件

## 降级策略

如果当前环境缺少某些工具（如无语义重排 API）：

1. ReAct 模式仍然可用（只需搜索 + 判断）
2. CodeAct 模式可降级为"搜索 → 规则 filter → dedupe → 报告"（跳过重排打分）
3. Gotcha 检测不依赖任何工具，永远可用——这是本 skill 的核心价值
