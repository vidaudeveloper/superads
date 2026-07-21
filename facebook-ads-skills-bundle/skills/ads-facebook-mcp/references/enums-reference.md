# 字段与枚举对照（enums-reference）— Facebook / Meta Ads

> **校准基线**：本技能枚举基于 **Meta Marketing API 官方枚举** 编写，并与 TikTok 同构 skill（`ads-tiktok-mcp`）保持结构对齐。
> VidAU 封装层可能对 Meta 字段做简化封装（同 TikTok 的「VidAU 简化值」模式）。**真正传给 VidAU Facebook MCP 的值，必须以真实 VidAU Facebook MCP（部署域名见 mcp-client §1）返回体 / 接受值为准校准**；下表「官方 Meta 值」为权威来源，VidAU 若有简化值需在校准后补充并列。
>
> ⚠️ 与 TikTok 的关键差异：
> 1. **层级名**：Meta 三层为 Campaign → **Ad Set** → Ad（TikTok 为 Campaign → Adgroup → Ad）。
> 2. **状态枚举**：Meta 用 **`ACTIVE` / `PAUSED`**（TikTok 用 `ENABLE` / `DISABLE`），**切勿混用**。
> 3. **货币**：Meta 金额多为字符串数字（如 `"120.50"`），币种取自 `currency` 字段（USD 等）。
> 4. **出价策略**：Meta 用 `bid_strategy`（不是 TikTok 的 `bid_type`）。
> 5. **版位**：Meta 用 `publisher_platforms` / `device_platforms` / `positions` 体系（不是 TikTok 的 `placement_type`）。

---

## 0. Python 常量（供 `validate_args` 导入 / 内联）

```python
# 本技能使用值 = Meta Marketing API 官方枚举（VidAU 若做简化封装，需在校准后并入）
# === Campaign 营销目标（objective，Meta 2022+ 简化版） ===
OBJECTIVES = {
    "AWARENESS", "TRAFFIC", "ENGAGEMENT", "LEADS",
    "APP_PROMOTION", "SALES",
}
# 兼容旧版官方枚举（部分账户/接口仍返回旧值）
OBJECTIVES |= {
    "BRAND_AWARENESS", "REACH", "APP_INSTALLS", "VIDEO_VIEWS",
    "LEAD_GENERATION", "MESSAGES", "CONVERSIONS", "CATALOG_SALES",
    "STORE_VISITS", "OUTCOME_ENGAGEMENT", "OUTCOME_AWARENESS",
    "OUTCOME_TRAFFIC", "OUTCOME_LEADS", "OUTCOME_APP_PROMOTION", "OUTCOME_SALES",
}

# === Ad Set 优化目标（optimization_goal） ===
OPTIMIZATION_GOALS = {
    "AD_RECALL_LIFT", "APP_INSTALLS", "ENGAGED_USERS", "EVENT_RESPONSES",
    "IMPRESSIONS", "LANDING_PAGE_VIEWS", "LEAD_GENERATION", "LINK_CLICKS",
    "OFFSITE_CONVERSIONS", "POST_ENGAGEMENT", "REACH", "THRUPLAY", "VALUE",
    "VISIT_PIXELS", "CONVERSATIONS", "IN_APP_VALUE", "PROFILE_VISIT",
    "MEANINGFUL_CALL_DURATION", "REMINDER_SETS",
}
OPTIMIZATION_GOALS |= {"APP_INSTALLS_AND_OFFSITE_CONVERSIONS"}

# === 计费事件（billing_event） ===
BILLING_EVENTS = {
    "APP_INSTALLS", "IMPRESSIONS", "LINK_CLICKS",
    "OFFSITE_CONVERSIONS", "POST_ENGAGEMENT", "THRUPLAY",
}

# === 出价策略（bid_strategy） — Meta 特有 ===
BID_STRATEGIES = {
    "LOWEST_COST_WITHOUT_CAP",   # 最低成本（无上限，原 Lowest Cost）
    "LOWEST_COST_WITH_BID_CAP",  # 最低成本 + 出价上限（Bid Cap）
    "LOWEST_COST_WITH_MIN_ROAS", # 最低成本 + 最低 ROAS
    "COST_CAP",                  # 成本上限（Cost Cap）
}

# === 购买方式（buying_type） ===
BUYING_TYPES = {"AUCTION", "RESERVED", "FIXED_PRICE"}

# === 状态枚举（status） — Meta 用 ACTIVE/PAUSED，与 TikTok 不同！ ===
ENTITY_STATUSES = {"ACTIVE", "PAUSED", "DELETED", "ARCHIVED"}
CAMPAIGN_LEARNING_STATUS = {"LEARNING", "LEARNING_COMPLETED", "ACTIVE", "FAILED"}

# === 版位平台（publisher_platforms） ===
PUBLISHER_PLATFORMS = {"facebook", "instagram", "audience_network", "messenger"}

# === 广告主账户状态（list_advertisers 返回） ===
ACCOUNT_STATUS = {"ACTIVE", "DISABLED", "UNSETTLED", "PENDING_REVIEW", "IN_GRACE_PERIOD"}
```

---

## 1. Campaign 营销目标（objective）— Meta 官方简化版

| 官方 Meta 值 | 旧版对应值 | 含义 | 适用场景 |
|--------------|-----------|------|---------|
| `AWARENESS` | `BRAND_AWARENESS` / `REACH` | 品牌认知 | 覆盖量、品牌曝光 |
| `TRAFFIC` | `TRAFFIC` | 流量 | 落地页访问、链接点击 |
| `ENGAGEMENT` | `POST_ENGAGEMENT` / `MESSAGES` | 互动 | 帖子互动、消息、视频观看 |
| `LEADS` | `LEAD_GENERATION` | 潜在客户 | 线索表单、即时表单 |
| `APP_PROMOTION` | `APP_INSTALLS` | 应用推广 | App 安装 / 激活 / 再互动 |
| `SALES` | `CONVERSIONS` / `CATALOG_SALES` | 销售 | 网站转化、电商目录销售、店铺销售 |

> ⚠️ **转化类目标前置条件**：选择 `SALES`（或旧 `CONVERSIONS`）前，必须确认 **Meta Pixel + Conversions API 已配置**且转化事件已绑定（见 ad-creation-flow §1）。未配置则先引导配置，再创建转化广告。
> `SALES` 目标下若使用商品目录，需已配置商品目录（Catalog）与动态广告（DPA）。

---

## 2. 预算模式（budget）— Meta 特有

Meta **不用 budget_mode 枚举**，而是用字段区分：

| 字段 | 含义 | 说明 |
|------|------|------|
| `daily_budget` | 日预算 | 每日花费上限（字符串数字，如 `"50.00"`） |
| `lifetime_budget` | 总预算 | 整个排期总花费上限 |
| 两者都缺省 | 不限预算 | Campaign 层级可不限，由 Ad Set 控制 |

- **CBO（Campaign Budget Optimization）/ Advantage+ Campaign Budget**：在 Campaign 层设预算，Meta 自动分配到各 Ad Set。字段 `budget_optimization` = `true`。
- **ABO（Ad Set Budget Optimization）**：每个 Ad Set 独立设预算。

---

## 3. Ad Set 优化目标（optimization_goal）

| 官方 Meta 值 | 含义 | 常见搭配 |
|--------------|------|---------|
| `REACH` | 覆盖 | AWARENESS |
| `AD_RECALL_LIFT` | 广告回想提升 | AWARENESS |
| `IMPRESSIONS` | 展示 | AWARENESS / ENGAGEMENT |
| `LINK_CLICKS` | 链接点击 | TRAFFIC |
| `LANDING_PAGE_VIEWS` | 落地页浏览 | TRAFFIC（需 Pixel） |
| `POST_ENGAGEMENT` | 帖子互动 | ENGAGEMENT |
| `THRUPLAY` | 视频完整播放 | ENGAGEMENT / VIDEO_VIEWS |
| `LEAD_GENERATION` | 线索 | LEADS（即时表单） |
| `OFFSITE_CONVERSIONS` | 站外转化 | SALES（需 Pixel + 事件） |
| `VALUE` | 转化价值 | SALES（需 Purchase value 优化） |
| `APP_INSTALLS` | 应用安装 | APP_PROMOTION |
| `CONVERSATIONS` | Messenger 对话 | ENGAGEMENT / MESSAGES |

> 传入 MCP 用官方 Meta 值；VidAU 若有简化值需校准后替换。

---

## 4. 计费事件（billing_event）

| 官方 Meta 值 | 含义 |
|--------------|------|
| `IMPRESSIONS` | 展示计费（CPM，最常见） |
| `LINK_CLICKS` | 链接点击计费（CPC） |
| `APP_INSTALLS` | 应用安装计费 |
| `POST_ENGAGEMENT` | 帖子互动计费 |
| `THRUPLAY` | 视频完整播放计费 |
| `OFFSITE_CONVERSIONS` | 站外转化计费 |

> Meta 默认绝大多数广告按 `IMPRESSIONs`（CPM）计费；`LINK_CLICKS` 计费仅在特定优化目标可用。

---

## 5. 出价策略（bid_strategy）与出价金额映射 — Meta 特有

| 出价策略 | bid_strategy | bid_amount | billing_event | 说明 |
|----------|-------------|-----------|---------------|------|
| 最低成本（无上限） | `LOWEST_COST_WITHOUT_CAP` | 不设 | 任意 | Meta 默认，系统自动出价，最大化量 |
| 最低成本 + 出价上限 | `LOWEST_COST_WITH_BID_CAP` | > 0（单次最高出价） | 任意 | 控制单次最高花费 |
| 成本上限 | `COST_CAP` | 目标成本 | `IMPRESSIONS` / `OFFSITE_CONVERSIONS` | 控制平均成本（目标 CPA） |
| 最低 ROAS | `LOWEST_COST_WITH_MIN_ROAS` | min_roas | — | 电商价值优化，设最低广告支出回报 |

**学习期（Learning Phase）**：
- 新 Ad Set 需在 **7 天内累计 50 次转化** 才能结束学习期进入稳定。
- 学习期内成本/表现波动属正常，**避免大改**（预算 / 受众 / 出价 / 创意大幅调整会重置学习）。
- 出价调整建议：单次不超过原值 20%–50%，至少观察 2 天再改。

---

## 6. 版位（placements）— Meta 体系

### placement_type
| 值 | 含义 |
|----|------|
| `automatic` | 自动版位（Advantage+ Placements，推荐） |
| `manual` | 手动版位（编辑版位） |

### publisher_platforms（手动版位用）
| 值 | 含义 |
|----|------|
| `facebook` | Facebook 动态 |
| `instagram` | Instagram 动态 |
| `audience_network` | Audience Network（第三方联盟） |
| `messenger` | Messenger |

### 常见版位组合（positions，手动版位用）
- Facebook：`feed`、`stories`、`reels`、`video_feeds`、`marketplace`、`right_hand_column`、`search`
- Instagram：`feed`、`stories`、`reels`、`explore`、`explore_home`
- Audience Network：`rewarded_video`、`instream_video`、`native`、`banner`、`interstitial`
- Messenger：`stories`、`inbox`、`sponsored_messages`

---

## 7. 受众定向维度（targeting）— Meta 特有

| 维度 | 字段 | 说明 |
|------|------|------|
| 地理位置 | `geo_locations` | 国家 / 地区 / 城市 / 邮编 |
| 年龄 | `age_min` / `age_max` | 13–65+ |
| 性别 | `genders` | `1`=全部 / `2`=男 / `3`=女 |
| 语言 | `locales` | 语言代码数组 |
| 兴趣 / 行为 | `interests` / `behaviors` | Core Audiences 核心 |
| 自定义受众 | `custom_audiences` | Custom Audiences ID 数组 |
| 类似受众 | `lookalike_audiences` | Lookalike Audiences ID 数组 |
| 连接对象 | `connections` / `friends_of_connections` | 已赞主页 / 好友 |
| 设备 | `device_platforms` / `user_device` / `user_os` | mobile / desktop + 机型 / 系统 |
| Advantage+ 受众 | `advantage_audience` | `true` 时启用自动受众扩展（推荐） |

> Meta 建议使用 Advantage+ Audience 自动扩展，宽定向 + 系统优化效果通常优于过度收窄。

---

## 8. 状态枚举（status）— 统一口径，禁止与 TikTok 混用

| 实体 | 有效过滤值 | 说明 |
|------|-----------|------|
| Campaign / Ad Set / Ad | `ACTIVE` / `PAUSED` | 运行中 / 暂停 |
| Campaign / Ad Set / Ad（删除态） | `DELETED` / `ARCHIVED` | 已删除 / 已归档 |

- ⚠️ **Meta 用 `ACTIVE` / `PAUSED`，不是 TikTok 的 `ENABLE` / `DISABLE`**。
- `list_advertisers` 的账户状态：`ACTIVE` / `DISABLED` / `UNSETTLED` / `PENDING_REVIEW` / `IN_GRACE_PERIOD`。
- **学习阶段状态**（Ad Set 级）：`LEARNING` / `LEARNING_COMPLETED` / `ACTIVE` / `FAILED`，需单独展示给用户。

---

## 9. 创意规格（Creative specs）— Meta 各版位推荐

| 版位 | 推荐比例 | 推荐分辨率 | 视频时长 | 文件大小 |
|------|---------|-----------|---------|---------|
| Facebook / Instagram Feed | 1:1 或 **4:5**（推荐） | ≥ 1080×1080 | 15s 内最佳，最长 240s | ≤ 4GB |
| Facebook / Instagram Stories | **9:16** | ≥ 1080×1920 | 15s 内最佳，最长 120s | ≤ 4GB |
| Facebook / Instagram Reels | **9:16** | ≥ 1080×1920 | 15s 内最佳，最长 90s | ≤ 4GB |
| Marketplace / Right Column | 1:1 | ≥ 1080×1080 | — | ≤ 4GB |
| Audience Network | 1:1 或 16:9 | ≥ 1080×1080 | — | ≤ 4GB |

- 视频码率建议 ≥ 高清，**声音**：Meta 推荐带字幕与音效（大多数用户静音浏览）。
- **Advantage+ Creative**：开启后系统自动裁剪 / 增强素材适配各版位。
- 图片格式：JPG / PNG；视频格式：MP4 / MOV。

---

## 10. 报表层级（level）与下钻降级

`get_daily_metrics` 的 `level`：`ACCOUNT` / `CAMPAIGN` / `ADSET` / `AD`。

- 对应 TikTok 的 `ADVERTISER` / `CAMPAIGN` / `ADGROUP` / `AD`（Meta 用 `ACCOUNT` 与 `ADSET`）。
- 经验上 VidAU 对多数账户可能仅可靠返回 `ACCOUNT` 级别；`CAMPAIGN`/`ADSET`/`AD` 可能为空。
- 调用策略：优先尝试下钻层级；**若返回空，降级为 `ACCOUNT` 聚合并在报告中注明「该账户暂不支持广告级下钻」**，不得编造明细。

---

## 11. 关键指标字段类型速查

| 字段 | 类型 | 取值说明 |
|------|------|---------|
| `accountId` / `advertiserId` | str | 广告账户 ID（过滤用） |
| `campaignsCount` / `adsetsCount` / `adsCount` | int/str | 可能为字符串，比较前 `int()` |
| `balance` / `amount_spent` / `spend` | str / null | 字符串金额；`null`=未同步 |
| `account_status` | str | `ACTIVE` / `DISABLED` 等 |
| `status`（campaign/adset/ad） | str | `ACTIVE` / `PAUSED` / `DELETED` / `ARCHIVED` |
| `daily_budget` / `lifetime_budget` | str | 字符串数字，如 `"50.00"` |
| `spend` / `impressions` / `clicks` / `ctr` / `cpc` / `cpm` | str / null | 字符串数值，需 `float()` |
| `conversions` / `purchase_roas` / `cost_per_action_type` | str / null | 转化与价值指标 |
| `id`（create_* 返回） | str | **本地 UUID**（非线上 ID） |
| `facebookCampaignId` 等 | str | **线上 Meta ID** |

---

## 12. 广告审核与拒审原因分类（Meta 广告政策）

审核状态来自同步数据。常见拒审原因分类：

| 类别 | 典型原因 |
|------|---------|
| 素材问题 | 误导性声明、夸大承诺、前后对比图违规、低质素材、文字占比过高（旧规）、版权音乐 |
| 落地页问题 | 落地页不可用、与广告不符、跳转异常、无联系方式、支付不安全 |
| 政策限制 | 禁止内容（酒精 / 烟草 / 成人 / 武器等需资质）、个人属性定向违规、误导性政治内容 |
| 资质缺失 | 特殊行业未提交资质（金融 / 医疗 / 博彩 / 电商等） |

> 拒审后可在 Meta 事件管理器 / 广告管理工具查看具体原因，修改后重新提交审核。
