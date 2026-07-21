# Source and validation notes — Facebook / Meta

This skill is normalized from the user-provided TikTok Agent knowledge base (the `Tiktok-Ads-MCP.zip` bundle and the `skill迭代2.0-tiktok.docx` spec), the AI助手 page usage scenario, and Vidau Facebook/Meta ad monitoring/report/ad-creation requirements. It is the **Facebook/Meta channel counterpart** of `vidau-tiktok-agent-knowledge`, keeping the same two-layer (knowledge + execution) architecture and the same 2.0 iteration spec (trigger keywords → field collection → missing-field follow-up → field validation → conflict detection → structured output).

Primary runtime scenario:
- User uploads this skill in the platform's management page.
- User opens the Vidau Facebook platform (默认域名 `https://facebook.vidau.ai/`，真实域名以部署为准).
- The site defaults to the AI助手 page.
- User enters questions in the AI助手 panel.
- The platform invokes this skill and displays the returned user-facing content in the AI助手 panel.

Strict validation principles:
- Return valid JSON only for AI助手 panel interactions.
- Include both routing fields and `assistant_display` so the platform can parse the response while the user sees clean content.
- Do not fabricate business results or performance metrics.
- Preserve unknown/blocked states when tool permissions, account source, authorization, sync state, backend configuration, BM qualification, Facebook Page binding, or scope are not known.
- Treat high-risk actions as confirmation-required.
- Use `https://facebook.vidau.ai/` as the AI助手 entry page (真实域名以部署为准).
- Use `https://facebook.vidau.ai/campaigns/ads` as the Facebook ad creation/ad list page entry point.
- Do not claim page navigation, data sync, ad creation, ad publishing, or backend action succeeded without a tool/backend result.

Recommended platform rendering behavior:
1. Parse the full JSON.
2. Render `assistant_display.markdown` in the AI助手 panel.
3. Render `assistant_display.cards`, `charts`, and `action_buttons` if supported.
4. Use `ui_actions` only for frontend actions such as opening a page or panel.
5. Use `tool_requests` only to trigger backend tools after permission and confirmation checks.
6. Never display raw JSON to ordinary users unless debug mode is enabled.

## 与 TikTok skill 的对照与差异点

本 skill 与 `vidau-tiktok-agent-knowledge` 同构，关键差异（必须区分，不可混用）：

| 维度 | Facebook / Meta | TikTok |
|------|----------------|--------|
| 三层结构 | Campaign → **Ad Set** → Ad | Campaign → Adgroup → Ad |
| 状态枚举 | `ACTIVE` / `PAUSED` | `ENABLE` / `DISABLE` |
| 营销目标 | `AWARENESS`/`TRAFFIC`/`ENGAGEMENT`/`LEADS`/`APP_PROMOTION`/`SALES` | `REACH`/`TRAFFIC`/`VIDEO_VIEWS`/`LEAD_GEN`/`WEB_CONVERSIONS` 等 |
| 出价策略 | `bid_strategy`（LOWEST_COST_WITHOUT_CAP 等） | `bid_type`（BID_TYPE_NO_BID 等） |
| 版位体系 | `publisher_platforms`（facebook/instagram/audience_network/messenger） | `placement_type`（TIKTOK/PANGLE 等） |
| 转化追踪 | Meta Pixel + Conversions API + Advanced Matching | TikTok Pixel + Events API |
| 自动优化 | Advantage+ 系列 | CBO / Maximum Delivery |
| 广告主页 | 必须绑定 Facebook 主页（page_id） | 不需要 |
| 创意主比例 | Feed 4:5 / Stories-Reels 9:16 | 9:16 为主 |
| 商务管理 | BM（Business Manager） | BC（Business Center） |
| 平台域名 | `facebook.vidau.ai`（待校准） | `tiktok.vidau.ai` |
| 鉴权 | `FACEBOOK_MCP_API_KEY` | `TIKTOK_MCP_API_KEY` |

## MCP 执行层（Vidau Agent 对话模式）

本 skill 在 Vidau Agent 对话中使用时（非 AI助手面板），必须配合 `ads-facebook-mcp` skill 一起加载。

- `ads-facebook-mcp` 提供 `list_advertisers`、`get_daily_metrics`、`create_campaign`、`create_adset`、`create_ad` 等 MCP 工具（具体数量与名称以该 skill 当前版本为准；v2.3.0 已知含上述工具，未暴露 `get_alerts` 时异常预警改用 `get_daily_metrics` + 本 skill 阈值规则）。
- 详细映射与枚举（Meta 官方值）见 SKILL.md 的「执行上下文说明」章节。
- 关键规则（对齐 v2.3.0）：两步滤波法、`sync_advertiser_data` 仅指定账户且数据陈旧时谨慎调用（15s 超时、失败不阻塞）、报表先下钻 `level=ADSET/AD` 空则降级 `level=ACCOUNT` 并注明、日期范围以账户时区为准、状态用 `ACTIVE`/`PAUSED`、层级用 Ad Set。

## 待校准项（部署后需以真实 VidAU Facebook MCP 接受值为准）

1. **平台域名**：`facebook.vidau.ai` 为同构假设，真实域名以 Vidau 部署为准（见 mcp-client §1）。
2. **枚举简化**：VidAU 封装层可能对 Meta 字段做简化（同 TikTok 的「VidAU 简化值」模式）；若发现真实接受值与官方枚举不同，需在 `enums-reference.md` 补充「VidAU 值」列并更新 `validate_args`。
3. **工具名**：`get_adsets`/`create_adset` 为按 Meta 层级习惯命名；若 VidAU 实际暴露的工具名不同，需校准 SKILL.md 与 mcp-client.md。
4. **返回字段**：`facebookCampaignId`/`facebookAdsetId`/`facebookAdId` 为线上 Meta ID 字段名的假设；若 VidAU 返回体不同，需校准字段速查表。
