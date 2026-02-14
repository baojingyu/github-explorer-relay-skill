---
name: github-explorer-relay-skill
description: >
  Deep-dive analysis of GitHub projects using Agent Browser (Zero-API cost).
  Triggered by phrases like "帮我看看这个项目", "分析一下 repo", "了解一下 XXX".
  Replaces paid search APIs with direct browser automation to explore architecture, community health, and competitors.
---

# GitHub Explorer Relay Skill — 零成本项目深度分析

> **Philosophy**: README 只是门面，真正的价值藏在 Issues、Commits 和社区讨论里。
> **Mode**: Agent Browser (无需 Search API Key)

## Workflow

[项目名] → [1. 定位 Repo (Agent Browser)] → [2. 多源采集] → [3. 分析研判] → [4. 结构化输出]

### Phase 1: 定位 Repo

**注意：本阶段禁止调用 `web_search`。必须使用 `agent-browser` 工具模拟用户搜索。**

1. **定位 GitHub 仓库**：
    - **操作**：调用 `agent-browser` 访问搜索引擎（推荐 Bing 或 Google）。
    - **指令示例**：`agent-browser(action='open', targetUrl='https://www.bing.com/search?q=<project_name>', profile='chrome')`
    - **目标**：从搜索结果页面提取第一个匹配 `github.com/{org}/{repo}` 格式的链接。

2. **社区情报检索**：
    - **操作**：继续使用 `agent-browser` 查找社区反馈。
    - **指令示例**：`agent-browser(action='open', targetUrl='https://www.bing.com/search?q=<project_name> review 评测 坑点 vs', profile='chrome')`
    - **目标**：收集前 3-5 个高价值的讨论链接（优先关注 V2EX, Reddit, 掘金, Hacker News）。

3. **获取基础信息**：
    - 确认 Repo URL 后，使用 `web_fetch` 抓取仓库主页，获取 README、Stars、Forks、License 和最近更新时间。

### Phase 2: 多源采集（并行）

以下来源**按需检查**，有则采集，无则跳过：

| 来源 | URL 模式 | 采集内容 | 建议工具 |
|---|---|---|---|
| **GitHub Repo** | `github.com/{org}/{repo}` | README、About、Contributors | `web_fetch` |
| **GitHub Issues** | `github.com/{org}/{repo}/issues?q=sort:comments` | Top 3-5 高热度/高质量 Issue | `agent-browser` (渲染动态列表) |
| **技术博客/文档** | Medium/Dev.to/官方文档 | 架构分析、核心概念 | `web_fetch` |
| **社区讨论** | Reddit/V2EX/X (Twitter) | 真实评价、槽点、竞品对比 | `agent-browser` (搜索定位 + 读取) |
| **中文社区** | 微信公众号/知乎/小红书 | 深度评测、避坑指南 | `content-extract` (必需) |

#### 抓取降级与增强协议 (Extraction Upgrade)

当遇到以下情况时，**必须**放弃 `web_fetch`，改用 `content-extract` Skill：
1. **域名限制**: `mp.weixin.qq.com`, `zhihu.com`, `xiaohongshu.com`。
2. **结构复杂**: 页面包含大量公式 (LaTeX)、复杂表格、或 `web_fetch` 返回的 Markdown 极其凌乱。
3. **内容缺失**: `web_fetch` 因反爬返回空内容或 Challenge 页面。

**调用方式**：
```bash
python3 skills/content-extract/scripts/content_extract.py --url <URL>
```

### Phase 3: 分析研判

基于采集数据进行逻辑推理：

- **项目阶段判定**: 早期实验 / 快速成长 / 成熟稳定 / 维护模式 / 停滞（依据 Commit 频率和 Issue 响应速度）。
- **精选 Issue 标准**: 评论数多、Maintainer 深度参与、暴露架构设计缺陷、或包含有价值的最佳实践讨论。
- **竞品识别**: 从 README 的 "Comparison" 章节、Issues 中的 "Alternative" 讨论以及搜索引擎的 "vs" 关键词结果中提取。

### Phase 4: 结构化输出

严格按以下模板输出，**每个模块都必须有实质内容或明确标注"未找到"**。

#### 排版规则（强制）

1. **标题链接**：格式必须为 `# [Project Name](https://github.com/org/repo)`。
2. **视觉分隔**：每个粗体标题（如 `**🎯 ...**`）前后必须各有一个空行。
3. **Telegram 空行修复（强制）**：在列表块（`-` 开头）的末尾与下一个标题之间，**必须**插入一行盲文空格 `⠀`（U+2800）。格式如下：

```markdown
- 列表最后一项

⠀
**下一个标题**
```

4. **拒绝空泛**：社区声量部分严禁使用 "评价很高"、"热度不错" 等模糊描述，必须引用具体内容。
5. **信息溯源**：所有引用的外部信息（推文、帖子、文章）都必须附上原始 URL。

#### 输出模板

```markdown
# [{Project Name}]({GitHub Repo URL})

**🎯 一句话定位**

{是什么、核心解决什么痛点}

**⚙️ 核心机制**

{技术原理/架构/实现方式。用人话讲清楚，不要直接复制 README。包含关键技术栈。}

**📊 项目健康度**

- **Stars**: {数量}  |  **Forks**: {数量}  |  **License**: {类型}
- **团队/作者**: {背景，个人开发者还是公司维护？}
- **Commit 趋势**: {最近活跃度 + 项目阶段判断}
- **最近动态**: {最近几条重要 Feature 或 Fix}

**🔥 精选 Issue**

{Top 3-5 高质量 Issue，每条包含标题、链接、核心讨论点。如无高质量 Issue 则注明。}

**✅ 适用场景**

{什么时候该用，适合什么样的业务规模}

**⚠️ 局限 / 避坑**

{什么时候别碰，已知性能瓶颈或维护问题}

**🆚 竞品对比**

{同赛道项目对比，差异点。每个竞品必须附链接}
- **vs [Project A](URL)** — 差异描述
- **vs [Project B](URL)** — 差异描述

**🌐 知识图谱**

- **DeepWiki**: {链接或"未收录"}
- **Zread.ai**: {链接或"未收录"}

**🎬 Demo / 体验**

{在线 Demo 链接，或"无"}

**📄 关联论文**

{arXiv 链接，或"无"}

**📰 社区声量**

**X/Twitter**

{具体引用推文内容摘要 + 链接}
- [@某用户](链接): "具体评价内容..."
{如未找到则注明"未找到相关讨论"}

**中文社区**

{具体引用帖子标题/内容摘要 + 链接}
- [知乎/V2EX: 帖子标题](链接) — 核心观点...
{如未找到则注明"未找到相关讨论"}

**💬 架构师视角**

{你的主观评价：技术选型建议、代码质量直觉、未来维护风险预测}
```

## Execution Notes

- **Search Policy**: **Strictly Prohibit** `web_search` tool. All retrieval operations must be performed via `agent-browser` automation (Agent Browser Relay).
- **Relay Strategy**: Use `agent-browser` to navigate to search engines (`bing.com`, `google.com`) and parse results manually.
- **Extraction**: Use `web_fetch` for static GitHub pages. Use `content-extract` for complex or anti-bot protected articles.
- **Output Language**: Output in **Chinese** (Simplified), but keep technical terms in English.
- **Link Integrity**: Ensure all generated links are valid and accessible.

## ⚠️ 输出自检清单（强制，每次输出前逐条核对）

输出报告前，**必须逐条检查以下项目**，全部通过才可发送：

- [ ] **标题链接**：`# [Project Name](GitHub URL)` 格式，可点击跳转
- [ ] **标题空行**：每个粗体标题（`**🎯 ...**`）前后各有一个空行
- [ ] **Telegram 空行**：每个列表块末尾与下一个标题之间有盲文空格 `⠀` 行（防止 Telegram 吞空行）
- [ ] **Issue 链接**：精选 Issue 每条都有完整 `[#号 标题](完整URL)` 格式
- [ ] **竞品链接**：每个竞品都附 `[名称](GitHub/官网链接)`
- [ ] **社区声量链接**：每条引用都有 `[来源: 标题](URL)` 格式
- [ ] **信息溯源**：所有外部引用都附原始链接

## Dependencies

本 Skill 依赖以下 OpenClaw 工具和 Skills：

| 依赖 | 类型 | 用途 |
|---|---|---|
| `agent-browser` | Skill | **核心检索工具**。用于访问搜索引擎（Agent Browser Relay）定位项目、查找社区讨论，以及动态页面渲染。 |
| `web_fetch` | 内置工具 | 静态网页内容抓取（GitHub 页面首选）。 |
| `content-extract` | Skill | 高保真内容提取（反爬站点、微信公众号、知乎的降级方案）。 |
```
