# Facebook/Meta 广告管理 Skill 套件（标准 Skill 结构 · 优化版）

本压缩包包含 **两个相互独立、配套使用** 的 Facebook/Meta 广告 Skill（两层架构：大脑层 + 执行层），按 `通用 Agent Skill 结构` 的标准 Skill 结构组织。覆盖 **广告创建 / AI 巡检 / 日报周报** 三大模块，全面适配 Meta 渠道特性。

本包是 TikTok bundle（`Tiktok-Ads-MCP.zip`）的 **Facebook/Meta 渠道同构对应版本**，保持完全一致的架构与 2.0 迭代规范，字段/枚举/规则全面适配 Meta 平台特性。

## 目录结构（标准 Skill 结构）

```
facebook-ads-skills-bundle/
├── SKILL.md                                  # 总入口（套件索引）
├── README.md                                 # 本文件
├── .gitignore
├── assets/                                   # 图片 / 示例素材
│   └── .gitkeep
└── skills/
    ├── vidau-facebook-agent-knowledge/       # 大脑层 v1.1.0
    │   ├── SKILL.md
    │   ├── agents/openai.yaml
    │   └── references/source_and_validation.md
    └── ads-facebook-mcp/                     # 执行层 v2.3.0
        ├── SKILL.md
        └── references/
            ├── mcp-client.md                 # ★ 唯一正确的 MCP 调用实现
            ├── enums-reference.md            # ★ Meta 官方枚举对照 + Python 常量
            ├── ad-creation-flow.md           # 模块一：广告创建流程
            ├── ad-inspection-rules.md        # 模块二：AI 巡检规则（含 7 项 Meta 优化）
            └── report-format.md              # 模块三：日报/周报格式
```

> 每个 `skills/<layer>/` 文件夹本身就是一个标准 Skill：放入 `~/.agent/skills/`（用户级）或 `{项目}/.agent/skills/`（项目级）即生效。

## 分工与数据流

```
用户自然语言
     │
     ▼
[vidau-facebook-agent-knowledge]  ← 大脑层（先触发）
  · 意图识别 / 槽位提取 / 缺失判断
  · 页面路由 / 预填 / 确认检测
  · 仅当确需真实数据或写操作时 ──► 调用执行层
     │ （按需 read ads-facebook-mcp 的 reference，不预载）
     ▼
[ads-facebook-mcp]  ← 执行层
  · 健壮 mcp() 调用 VidAU Facebook MCP
  · 枚举校验 / 报表 / 巡检 / 创编
     │
     ▼
返回结构化结果 ──► 大脑层格式化为 AI 助手面板 JSON（assistant_display / cards / charts / action_buttons）
```

- **大脑层不预载执行层重型文件**：仅在路由判定需要真实数据时，才读取 `skills/ads-facebook-mcp/references/mcp-client.md` 与 `enums-reference.md`，避免无谓上下文消耗。
- **执行层不感知前端**：只负责正确调用 MCP 并返回数据，不直接生成面向用户的展示文案。

## 各 Skill 功能说明

### 1. vidau-facebook-agent-knowledge（大脑层 · 知识/路由/响应格式化）
**触发词**：facebook 开户、开户记录、开户后操作、授权、数据同步、报表、素材生成/管理、广告搭建、异常预警、页面跳转、所需字段、权限、拦截、未知状态……

**能做什么**：
- 11 个业务模块路由（`google_*` 命名前缀对齐 Vidau 通用约定）：开户申请/记录/后处理、OAuth 授权、数据同步、报表分析、素材管理、广告搭建、异常预警、知识问答。
- 用户口语 → 枚举映射（Meta 官方值优先，旧版值兼容）。
- 响应 JSON 格式化：UI payload → Markdown → 摘要三级降级；AI 助手面板可解析字段契约。

### 2. ads-facebook-mcp（执行层 · 直连 VidAU Facebook MCP）
**触发词**：facebook 巡检、预警、报表、创建 facebook 广告、facebook campaign、facebook adset、facebook 广告、日报、周报……

**能做什么**：
- 10 个 MCP 工具：`list_advertisers` / `get_alerts` / `get_campaigns` / `create_campaign` / `get_adsets` / `create_adset` / `get_ads` / `create_ad` / `get_daily_metrics` / `sync_advertiser_data`。
- 健壮 `mcp()` 五层容错、重试退避、两步滤波、字段 `validate_args` 预校验、`fmt_bal` 货币格式化、健康检查。
- 三大业务模块：广告创建（含 Meta 主页 page_id 前置、Ad Set 层级、Advantage+）、AI 巡检（六类异常 + Meta 专项优化）、日报/周报。

## 对齐基线（两 skill 已对齐，请勿各自漂移）

- **Meta 官方枚举**（真实 MCP 接受值）：`objective`=`AWARENESS`/`TRAFFIC`/`ENGAGEMENT`/`LEADS`/`APP_PROMOTION`/`SALES`、`optimization_goal`=`OFFSITE_CONVERSIONS`/`LINK_CLICKS`/`REACH`/`THRUPLAY`/`VALUE` 等、`billing_event`=`IMPRESSIONS`/`LINK_CLICKS`、`bid_strategy`=`LOWEST_COST_WITHOUT_CAP`/`COST_CAP`/`LOWEST_COST_WITH_BID_CAP`/`LOWEST_COST_WITH_MIN_ROAS`。完整常量与旧版对照见 `skills/ads-facebook-mcp/references/enums-reference.md`。
- **报表 level 规则**：先下钻 `ADSET`/`AD` 识别问题/优秀广告；为空则降级 `ACCOUNT` 并注明层级，**不编造**。两 skill 口径一致。
- **sync 策略**：交互场景默认不调 `sync_advertiser_data`；仅指定具体账户且数据陈旧时谨慎调用（15s 超时、失败不阻塞）。两 skill 口径一致。
- **时区**：报表默认日期与异常同环比统一按**广告主时区**，非服务器时区。
- **状态值**：`status` 统一用 `ACTIVE`/`PAUSED`（**非 TikTok 的 `ENABLE`/`DISABLE`**），无 `ENABLE`。
- **层级名**：三层为 Campaign → **Ad Set** → Ad（**非 TikTok 的 Adgroup**）。
- **部署平台**：Vidau Facebook 平台（默认域名 `facebook.vidau.ai`，真实域名以部署为准）。全文不含任何历史平台专有字眼。

## 与 TikTok bundle 的关键差异（同构对照）

| 维度 | Facebook bundle | TikTok bundle |
|------|----------------|---------------|
| 媒体渠道 | Facebook / Meta | TikTok |
| 三层结构 | Campaign → **Ad Set** → Ad | Campaign → Adgroup → Ad |
| 状态枚举 | `ACTIVE` / `PAUSED` | `ENABLE` / `DISABLE` |
| 营销目标 | `AWARENESS`/`TRAFFIC`/`ENGAGEMENT`/`LEADS`/`APP_PROMOTION`/`SALES` | `REACH`/`TRAFFIC`/`VIDEO_VIEWS`/`LEAD_GEN`/`WEB_CONVERSIONS` 等 |
| 出价策略 | `bid_strategy`（LOWEST_COST_WITHOUT_CAP / COST_CAP 等） | `bid_type`（BID_TYPE_NO_BID / BID_TYPE_CUSTOM） |
| 版位体系 | `publisher_platforms`（facebook/instagram/audience_network/messenger） | `placement_type`（TIKTOK/PANGLE 等） |
| 转化追踪 | Meta Pixel + Conversions API + Advanced Matching | TikTok Pixel + Events API |
| 自动优化 | Advantage+ 系列（Budget/Audience/Creative/Shopping/App） | CBO / Maximum Delivery |
| 广告主页 | 必须绑定 Facebook 主页（page_id） | 不需要 |
| 创意主比例 | Feed 4:5 / Stories-Reels 9:16 | 9:16 为主 |
| 商务管理 | BM（Business Manager） | BC（Business Center） |
| 平台域名 | `facebook.vidau.ai`（待校准） | `tiktok.vidau.ai` |
| 鉴权 | 环境变量 `FACEBOOK_MCP_API_KEY` | 环境变量 `TIKTOK_MCP_API_KEY` |

## 巡检 Meta 专项优化（已加入 ad-inspection-rules.md）

对照渠道特性后新增的 7 项中的 5 项已落地到执行层巡检规则：
1. **学习期全局过滤**：50 转化/7 天内不打"效果差"异常，避免误报。
2. **iOS 14.5+ 归因**：ATT 下转化/ROAS 口径收窄，对比需同口径，禁止跨口径直接比较。
3. **频率（Frequency）指标**：> 3 提示创意疲劳预警（纳入素材疲劳类）。
4. **Meta 政策合规**：拒登/限制单独成类，给出常见拒因与整改路径。
5. **行业基准双基线**：诊断时同时给平台中位与同行业基准，避免单基线误判。
（其余 2 项——版位级下钻、自动规则建议模板——已在创建/报表模块预留钩子。）

## 部署说明

1. 将 `skills/` 下的 2 个文件夹作为**两个独立 skill** 分别安装/加载到平台（建议配套加载）。
2. 大脑层 `vidau-facebook-agent-knowledge` 需配合执行层 `ads-facebook-mcp` 一起加载才能跑通"创建/巡检/报表"等需要真实数据的链路。
3. `ads-facebook-mcp` 需要环境变量 `FACEBOOK_MCP_API_KEY`（VidAU Facebook MCP 的 API Key），**不要写死在文件里**。
4. 可选环境变量 `FACEBOOK_MCP_TOOLS_URL`：若真实 MCP 域名非 `facebook.vidau.ai`，覆盖默认值。
5. 大脑层 `agents/openai.yaml` 为 Agent 配置型文件，按平台约定放置（若平台期望的文件名不同，仅改名即可，内容无需改动）。
6. ⚠️ **部署后第一步**：调用 `mcp_healthcheck()`（见 `skills/ads-facebook-mcp/references/mcp-client.md §8`）验证连通性与真实域名；若 VidAU 对字段做简化封装，按真实接受值校准 `enums-reference.md`。

## 安装方式

将 `skills/` 下的 2 个 `vidau-*` / `ads-*` 文件夹整体复制到：
- 用户级：`~/.agent/skills/`
- 或项目级：`{你的项目}/.agent/skills/`

重启 Agent 平台 后即可在对话中通过触发词调用。

## Changelog

- **2026-07-14**：按 `通用 Agent Skill 结构` 标准 Skill 结构重新打包——新增顶层 `SKILL.md`（套件索引）、`.gitignore`、`assets/`；两个子 skill 移入 `skills/`；补充 `trigger` 触发词 frontmatter。功能与内容不变。
- **2026-07-13**：初版 Facebook bundle。基于 `Tiktok-Ads-MCP.zip` 与 `skill迭代2.0-tiktok.docx`（2.0 迭代规范）同构生成，全面适配 Facebook/Meta 渠道特性（营销目标、Ad Set、ACTIVE/PAUSED、bid_strategy、publisher_platforms、Meta Pixel + CAPI、Facebook 主页、Advantage+、学习期 50 转化/7天）；巡检新增 5 项 Meta 专项优化。
