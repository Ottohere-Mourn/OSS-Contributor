# OSS-Contributor

> 一句话：告诉它你想在哪个技术领域做开源贡献，它帮你找到最合适的机会，生成审查报告，并在你确认后自动提交 PR。

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](./LICENSE)
[![Claude Code Skill](https://img.shields.io/badge/Claude%20Code-Skill-blue)](https://code.claude.com/docs/en/skills)

<p align="center">
  <img src="assets/image.png" alt="OSS-Contributor Banner" width="100%">
</p>

---

## 为什么需要这个工具

在 LLM/AI 方向的技术求职中，给知名开源项目提交高质量 PR 是极具说服力的技术名片。但从"我想搞 LLM 方向"到真正提交一个会被合并的 PR，中间有大量繁琐工作：

- 翻 GitHub Trending、awesome 列表、社区讨论，手动筛选活跃仓库
- 逐个阅读 CONTRIBUTING.md，判断项目是否对新人友好
- 在几百个 issue 里找"适合自己水平 + 有合入概率 + 对项目有价值"的那个
- 做完之后缺乏结构化记录，面试时说不清楚自己的贡献思路

**OSS-Contributor 把发现和评估自动化了，但把最终决策和执行确认留给了你。**

---

## 快速开始

### 前置条件

```bash
# 确保已安装并登录 GitHub CLI
gh auth login

# 验证
gh auth status
```

### 安装

```bash
# 直接克隆到 Claude Code skills 目录
git clone https://github.com/Ottohere-Mourn/OSS-Contributor.git ~/.claude/skills/oss-contributor

# 编辑偏好配置（可选）
vim ~/.claude/skills/oss-contributor/config.yaml
```

### 使用

在 Claude Code 中直接说：

> "帮我找一些 LLM 推理优化方向的开源项目，我想提交 PR"

或者使用斜杠命令：

```bash
/oss-contribute "LLM inference" --topk 3 --type docs,bugfix
```

或者用自然语言描述需求——模型会自动识别并调用此 Skill。

**PR 提交后，跟进 review 反馈：**

```bash
/oss-contribute followup --session 20260530-143000
```

---

## 工作流程

```
输入：领域关键词（如 "LLM inference optimization"）
  │
  ▼
┌──────────────────────────────────────────────┐
│ 1. 发现（Discover）                           │
│    多维度发散搜索 → 交叉验证 → 评分排序       │
│    人类在环：确认搜索维度                     │
│    输出: repos.json                           │
└──────────────────────────────────────────────┘
  │
  ▼
┌──────────────────────────────────────────────┐
│ 2. 评估（Evaluate）                           │
│    深度分析 Top-K 仓库：代码审计 + 机会矩阵    │
│    输出: opportunities.md                     │
└──────────────────────────────────────────────┘
  │
  ▼
┌──────────────────────────────────────────────┐
│ 3. 审查（Review）                             │
│    人类审查机会报告 → 选择目标                 │
│    输出: selection.json                       │
└──────────────────────────────────────────────┘
  │
  ▼
┌──────────────────────────────────────────────┐
│ 4. 执行（Execute）                            │
│    Worktree 隔离 → TDD → PR 提交              │
│    失败自动重试 + 跨 Agent 教训传递            │
│    输出: pr-links.json                        │
└──────────────────────────────────────────────┘
  │
  ▼
┌──────────────────────────────────────────────┐
│ 5. 报告（Report）                             │
│    结构化贡献报告，面试可直接使用               │
│    输出: contribution-report.md               │
└──────────────────────────────────────────────┘
  │
  ▼
┌──────────────────────────────────────────────┐
│ 6. 跟进（Followup）                           │
│    /oss-contribute followup --session <id>    │
│    检查 PR 状态 + 处理 review 反馈             │
└──────────────────────────────────────────────┘
```

---

## 与其他工具的区别

| | OSS-Contributor | Sweep AI | auto-github-contributor | autonomous-dev-team |
|---|---|---|---|---|
| 领域发现 | ✅ 多维度发散搜索 | ❌ | ❌ | ❌ |
| 人类在环审查 | ✅ 两个决策 gate | ❌ | 部分 | ❌ |
| 代码级评估 | ✅ shallow clone 验证 | ❌ | ❌ | ❌ |
| 结构化报告 | ✅ 面试可用 | ❌ | ❌ | ❌ |
| PR 跟进 | ✅ followup 命令 | ❌ | ❌ | ❌ |
| 自动合入 | ❌ 有意不做 | ✅ | ❌ | ✅ |

**关键设计哲学**：OSS-Contributor 帮你加速"发现"和"评估"，但把"决策"和"提交"的控制权完全留给你。它不会自动合入任何代码。

---

## 配置

编辑 skill 目录下的 `config.yaml`：

```yaml
defaults:
  topk: 5                  # 深度评估的仓库数
  min_stars: 500           # 最低 star 数
  languages: ["Python"]    # 语言偏好（空数组 = 不限）
  types: ["bugfix", "docs", "test"]  # 贡献类型偏好
  exclude_repos:           # 永远跳过的仓库
    - "tensorflow/tensorflow"

profile:
  name: "Your Name"
  github_username: "Ottohere-Mourn"

search:
  max_pages: 3
  sort_by: "stars"
  min_updated_days: 180

execution:
  max_iterations: 20
  require_human_gate: true
```

命令行参数会覆盖配置文件。

---

## 安全边界

- **绝不** force push、直接提交 main/master、跳过 pre-commit hook
- **人类 Gate 不可绕过**：Phase 3 审查环节不能用 `--auto` 跳过
- **Worktree 隔离**：每个 PR 在独立环境中操作，互不污染
- **不做凭证管理**：认证寄生在 `gh auth` 上，Skill 不接触 token
- **跟进命令不自动评论**：除了"感谢，已修改"之外，不会在 PR 下擅自发言

---

## 文件结构

```
oss-contributor/
├── SKILL.md                    # 主编排 Skill（6 个 Phase）
├── config.yaml                 # 用户偏好配置
├── prompts/
│   ├── discover.md             # Phase 1: 多维度搜索 + 评分
│   ├── evaluate.md             # Phase 2: 深度评估 + 代码验证
│   ├── execute.md              # Phase 4: 隔离执行 + TDD
│   ├── followup.md             # Phase 6: PR 跟进
│   └── report.md               # Phase 5: 报告生成
└── templates/
    ├── repos-schema.json        # 仓库数据结构
    ├── opportunities-schema.json # 机会数据结构
    └── report-template.md       # 报告模板
```

---

## 相关工具

以下项目在本 Skill 的设计过程中提供了有价值的参考：

- [auto-github-contributor](https://github.com/nexu-io/auto-github-contributor) — 执行流程的参考设计
- [autonomous-dev-team](https://github.com/zxkane/autonomous-dev-team) — Worktree 隔离模式
- [WhatIfWeDigDeeper/agent-skills](https://github.com/WhatIfWeDigDeeper/agent-skills) — Skill 工程化实践
- [Sweep AI](https://sweep.ai/) — Issue-to-PR 自动化的先行者

---

## 许可证

本项目采用 MIT 许可证 —— 详见 [LICENSE](./LICENSE) 文件。简而言之：你可以自由使用、修改、分发本代码，只需保留原始版权和许可声明。
