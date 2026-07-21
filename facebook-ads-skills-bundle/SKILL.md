---
name: facebook-ads-toolkit
description: Facebook/Meta 广告管理 Skill 套件（Vidau 平台，两层架构）。大脑层 vidau-facebook-agent-knowledge 负责 AI 助手知识/路由/响应格式化；执行层 ads-facebook-mcp 直连 VidAU Facebook MCP 做广告巡检、Campaign/Ad Set/Ad CRUD、日报周报。含广告创建、AI 巡检、日报/周报三大模块，全面适配 Meta 渠道特性（学习期、iOS 14.5+ 归因、频率、政策合规、行业基准）。配套两个标准 Skill，放入 ~/.agent/skills/ 或 {项目}/.agent/skills/ 即生效。
version: 1.0.0
category: advertising
tags: [facebook, meta, ads, mcp, vidau, inspection, reporting, toolkit]
---

# Facebook/Meta 广告管理 Skill 套件（Vidau）

**Vidau Facebook 平台两层式 AI 广告 Skill 套件**：大脑层负责知识/路由/响应格式化，执行层直连 VidAU Facebook MCP（`FACEBOOK_MCP_API_KEY`）做真实数据读写。覆盖 **广告创建 / AI 巡检 / 日报周报** 三大模块，全面适配 Meta 渠道特性（Campaign → Ad Set → Ad、状态 `ACTIVE`/`PAUSED`、`bid_strategy`、版位 `publisher_platforms`、Meta Pixel + CAPI、Facebook 主页前置、Advantage+、学习期 50 转化/7 天、iOS 14.5+ 归因、频率、政策合规、行业基准）。

## 技能文件结构

```
facebook-ads-skills-bundle/
├── SKILL.md                                  # 总入口：本文件（套件索引）
├── README.md                                 # 套件说明与安装方式
├── .gitignore
├── assets/                                   # 图片 / 示例素材
│   └── .gitkeep
└── skills/
    ├── vidau-facebook-agent-knowledge/       # 大脑层 v1.1.0（知识/路由/响应格式化）
    │   ├── SKILL.md
    │   ├── agents/openai.yaml
    │   └── references/source_and_validation.md
    └── ads-facebook-mcp/                     # 执行层 v2.3.0（直连 VidAU Facebook MCP）
        ├── SKILL.md
        └── references/
            ├── mcp-client.md                 # ★ 唯一正确的 MCP 调用实现
            ├── enums-reference.md            # ★ Meta 官方枚举对照 + Python 常量
            ├── ad-creation-flow.md           # 模块一：广告创建流程
            ├── ad-inspection-rules.md        # 模块二：AI 巡检规则
            └── report-format.md              # 模块三：日报/周报格式
```

## 各 Skill 功能速览

| Skill | 触发词 | 核心能力 |
|-------|--------|----------|
| `vidau-facebook-agent-knowledge` | facebook 开户 / 开户记录 / 授权 / 报表 / 巡检 / 素材 / 广告搭建 / 异常预警 / 页面跳转 / 字段 / 权限 / 拦截 | 意图识别、槽位提取、页面路由、AI 助手面板 JSON 响应格式化（UI payload → Markdown → 摘要三级降级）、11 个业务模块路由 |
| `ads-facebook-mcp` | facebook 巡检 / 预警 / 报表 / 创建 facebook 广告 / facebook campaign / adset / 广告 / 日报 / 周报 | 直连 VidAU Facebook MCP：10 个工具、健壮 `mcp()` 容错、枚举校验、广告创建/巡检/报表 |

## 安装方式

将 `skills/` 下的 2 个 `vidau-*` / `ads-*` 文件夹整体复制到：
- 用户级：`~/.agent/skills/`
- 或项目级：`{你的项目}/.agent/skills/`

重启 Agent 平台 后即可在对话中通过触发词调用。**大脑层与执行层需配套加载**才能跑通需要真实数据的链路。
