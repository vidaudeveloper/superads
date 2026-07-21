# 广告创建流程（ad-creation-flow）

本文件为「模块一：广告创建」的详细流程规范。所有 MCP 调用遵循 `references/mcp-client.md`，鉴权 key 取自环境变量 `FACEBOOK_MCP_API_KEY`。

> 本流程与 `ads-tiktok-mcp` 的创建流程同构，差异点已标注「Meta 特有」。

---

## 1. 投放前检查（不通过不得提交）

1. 账户授权状态 — 是否已授权、有效（`list_advertisers` 的 `account_status` 为 `ACTIVE`）
2. 账户余额 — 是否充足（`balance` 非 `null` 且 > 0）
3. 营销目标（objective）— 是否已选择（枚举见 enums-reference §1）
4. 优化目标（optimization_goal）— 是否已选择（枚举见 enums-reference §3）
5. **Facebook 主页**（page_id）— **Meta 特有**：Meta 广告必须绑定主页，确认主页已绑定且有发布权限
6. 素材 — 是否已上传且审核通过（**MCP 暂无上传接口**，见下方兜底）
7. 定向 — 地理位置、年龄、性别、兴趣、行为、自定义受众、类似受众、设备、版位等（见 enums-reference §7）
8. 预算 — 日预算（daily_budget）/ 总预算（lifetime_budget）/ CBO
9. 出价 — 出价策略与金额（映射见 enums-reference §5）
10. **转化追踪前置**（转化类目标专用）— `SALES` / `OFFSITE_CONVERSIONS` / `VALUE` 目标前，确认 **Meta Pixel + Conversions API 已配置**且转化事件已绑定

**不通过**：不允许进入创建提交，必须提示原因（如「账户未授权」「余额不足」「营销目标未选」「素材未上传」「未绑定 Facebook 主页」「转化目标需先配置 Pixel+CAPI」）。

---

## 2. 参数收集顺序

按面板依次收集：**广告账户 → 营销目标 → 优化目标 → 素材 → 主页 → 定向 → 预算 → 出价**。

> 与 TikTok 相比，Meta 多了「主页（page_id）」这一必填项，位于素材之后、定向之前。

### 预算检测
当用户选择同一广告账户 + 同一营销目标 + 同一优化目标时，检测该组合 **7 日内预算平均值**。当前输入超/低于均值阈值范围时，提示建议预算区间（如「近 7 日同类广告组日均预算 $45±$15，建议填写 $30–$60」）。

### 素材兜底（关键）
- `create_ad` 需要 `creative_id` 或 `video_id`，但 **MCP 当前无上传素材工具**。
- 若用户尚未上传素材：明确提示「需要先上传素材」，并给出两种方式：
  1. 打开面板在 VidAU 上传（`打开面板` 指令，见 SKILL.md）；
  2. 若环境支持，fallback 浏览器上传。
- **禁止**在素材缺失时强行提交创建。
- 素材规格建议（见 enums-reference §9）：
  - Feed：1:1 或 4:5（推荐 4:5），≥ 1080×1080
  - Stories / Reels：9:16，≥ 1080×1920
  - 视频时长 15s 内最佳
- **Advantage+ Creative** 建议开启：系统自动裁剪 / 增强素材适配各版位。

### 主页兜底（Meta 特有，关键）
- Meta 广告必须绑定 Facebook 主页（`page_id`）。
- 若用户未绑定主页或主页无发布权限：明确提示「需要先绑定 Facebook 主页并确认发布权限」。
- 可选绑定 Instagram 账号（`instagram_actor_id`）用于 Instagram 版位。
- **禁止**在主页缺失时强行提交创建。

### 转化追踪兜底（转化类目标专用）
- 选择 `SALES` / `CONVERSIONS` 目标或 `OFFSITE_CONVERSIONS` / `VALUE` 优化目标前：
  - 确认 **Meta Pixel 已安装**在落地页。
  - 确认 **Conversions API (CAPI)** 已配置（建议 Pixel + CAPI 双通道，提升事件量与归因质量）。
  - 确认 **转化事件已绑定**（如 Purchase、Add to Cart、Lead 等）。
  - 建议开启 **Advanced Matching**（Manual + Automatic），提升用户匹配与归因。
- 未配置时：先引导配置（路由到 `facebook_authorization` 或提示平台操作），再创建转化广告。

---

## 3. MCP 工具映射

| 步骤 | 工具 | 说明 |
|------|------|------|
| 检查账户 | `list_advertisers` | 授权 / 余额 / campaignsCount |
| 创建 Campaign | `create_campaign` | 返回本地 UUID（`facebookCampaignId` 为线上 ID） |
| 创建 Ad Set | `create_adset` | `campaign_id` 用本地 UUID |
| 创建 Ad | `create_ad` | `adset_id` 用本地 UUID；需素材 ID + 主页 ID |

---

## 4. 创建完成处理

| 状态 | 处理 |
|------|------|
| 成功 | 返回 Campaign/Ad Set/Ad 的本地 ID + 线上 ID；触发 AI 巡检询问（见 SKILL 场景 1） |
| 失败 | 自动重试最多 3 次（`mcp_retry`）→ 仍失败返回失败原因 + 建议修改字段 |
| 进行中 | 明确提示「创建进行中，等待接口返回」，不默认认为成功 |

**成功后自动触发巡检询问**（内容见 SKILL「巡检自动触发条件 - 场景 1」）。

> Meta 建议：新建 Ad Set 初始状态设为 `PAUSED`，确认无误后改 `ACTIVE` 启动；启动后进入学习期（7 天 / 50 次转化），期间避免大改。

---

## 5. 输出格式要求

- 不得编造任何广告创建结果、Meta 返回数据或广告状态。
- 接口未返回结果时，明确提示当前状态，而非默认成功。
- 所有创建 / 修改 / 发布等高风险操作，均需用户明确确认后方可执行。

---

## 6. 异常处理

| 异常类型 | 处理规则 |
|----------|----------|
| 通用异常 | 优先返回明确原因（由 `mcp()` 把 MCP 错误翻译为中文），不直接透传原始错误码 |
| MCP / Meta 接口异常 | `mcp_retry` 自动重试最多 3 次 |
| 网络异常 | 超时重请求一次；连续失败提示网络异常 |
| 用户取消 | 「取消」「不用了」「退出」→ 立即中止创建，返回「已取消广告创建」；若已成功提交 Meta，返回「本次创建已成功提交，无法终止」 |
| 广告审核被拒 | 返回拒审原因分类（见 enums-reference §12：素材 / 落地页 / 政策 / 资质），建议修改方向 |

---

## 7. Meta 创建流程要点（与 TikTok 差异速查）

| 维度 | Facebook / Meta | TikTok |
|------|----------------|--------|
| 层级 | Campaign → Ad Set → Ad | Campaign → Adgroup → Ad |
| 必填主页 | ✅ page_id | ❌ |
| 转化追踪 | Meta Pixel + Conversions API + Advanced Matching | TikTok Pixel + Events API |
| 出价 | bid_strategy（LOWEST_COST_WITHOUT_CAP 等） | bid_type（BID_TYPE_NO_BID 等） |
| 预算字段 | daily_budget / lifetime_budget | budget + budget_mode |
| 版位 | publisher_platforms + placement_type | placement_type + placements |
| 自动优化 | Advantage+（Budget/Audience/Creative/Shopping） | CBO / Maximum Delivery |
| 状态 | ACTIVE/PAUSED | ENABLE/DISABLE |
| 创意主比例 | Feed 4:5 / Stories-Reels 9:16 | 9:16 为主 |
