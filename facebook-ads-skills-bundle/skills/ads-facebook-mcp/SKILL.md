---
name: ads-facebook-mcp
description: 直连 VidAU Facebook/Meta MCP 接口——无需登录、最快路径。用 execute_code + requests 发 JSON-RPC 到 /api/mcp/tools，支持广告巡检、预警、Campaign/Ad Set/Ad CRUD、日报/周报。常规操作走 MCP；仅面板打开、素材上传、Meta OAuth 这类 MCP 做不到的才 fallback 浏览器。
version: 2.3.0
category: advertising
tags: [facebook, meta, ads, mcp, vidau, reporting, inspection]
trigger: facebook 巡检、facebook 预警、facebook 报表、创建 facebook 广告、facebook campaign、facebook adset、facebook 广告、facebook 日报、facebook 周报、facebook 指标
---

# Facebook Ads MCP — 直连 HTTP（最快路径）

**无需登录、无需 Python mcp 包。** 用 `execute_code` + `requests` 直接 POST JSON-RPC 到 VidAU Facebook MCP tools 端点。

> 本技能是 **VidAU MCP 的封装**（VidAU 再封装 Facebook/Meta Marketing API）。**字段/枚举以真实 VidAU Facebook MCP 返回体 / 接受值为准（基于 Meta Marketing API 官方枚举校准）**，权威对照见 `references/enums-reference.md`（该文件同时给出可直接导入的 Python 常量，供调用前 `validate_args` 预校验；校验同时放行旧版官方值，确保不报错）。
>
> ⚠️ **平台域名**：默认 `https://facebook.vidau.ai/api/mcp/tools`（VidAU 同构假设）。**真实域名需在首次部署时确认**（见 mcp-client §1）；环境变量 `FACEBOOK_MCP_API_KEY`。

**应用场景**：广告巡检、查预警、看报表、列出 Campaign/Ad/Ad Set、同步数据、广告创建。

### 技能文件结构

```
ads-facebook-mcp/
├── SKILL.md                     # 总入口：调用规范、工具清单、业务模块、渲染/定时/坑
└── references/
    ├── mcp-client.md            # ★ 唯一正确的 MCP 调用实现（容错/重试/滤波/校验/健康/排查）
    ├── enums-reference.md       # ★ Meta 官方枚举对照 + 可直接导入的 Python 常量
    ├── ad-creation-flow.md      # 模块一：广告创建流程
    ├── ad-inspection-rules.md   # 模块二：AI 巡检规则
    └── report-format.md         # 模块三：日报/周报格式
```

> 任何模块需要调用 MCP，都必须 `from references.mcp_client import mcp, mcp_retry, validate_args`（或内联其实现），禁止重复造轮子。

---

## 前置条件

```bash
# 在宿主环境或 execute_code 中预设（切勿明文写入本文件）
export FACEBOOK_MCP_API_KEY="fb_xxx"
# 可选：若真实域名不同，覆盖默认
export FACEBOOK_MCP_TOOLS_URL="https://facebook.vidau.ai/api/mcp/tools"
```

所有调用逻辑（健壮 `mcp()`、重试、两步滤波、时区、字段类型、参数预校验、健康检查）统一在 **`references/mcp-client.md`**，各模块不得自行实现请求。

> 会话开始建议先调用 `mcp_healthcheck()`（见 mcp-client §8）验证 key 与连通性，早失败早提示。

---

## 核心调用（速览，完整实现见 mcp-client.md）

```python
import requests, json, os

API_KEY = os.environ.get("FACEBOOK_MCP_API_KEY", "")
TOOLS_URL = os.environ.get("FACEBOOK_MCP_TOOLS_URL", "https://facebook.vidau.ai/api/mcp/tools")

def mcp(name, args=None, timeout=15):
    if not API_KEY:
        raise RuntimeError("缺少 FACEBOOK_MCP_API_KEY")
    payload = {"jsonrpc":"2.0","method":"tools/call",
               "params":{"name":name,"arguments":args or {}},"id":1}
    try:
        r = requests.post(f"{TOOLS_URL}?apiKey={API_KEY}", json=payload,
                          headers={"Content-Type":"application/json"}, timeout=timeout)
    except requests.exceptions.Timeout:
        raise RuntimeError(f"MCP 请求超时（>{timeout}s）：{name}")
    except requests.exceptions.RequestException as e:
        raise RuntimeError(f"MCP 网络错误：{e}")
    try:
        resp = r.json()
    except ValueError:
        raise RuntimeError(f"MCP 返回非 JSON（HTTP {r.status_code}）")
    if "error" in resp:
        err = resp["error"] or {}
        raise RuntimeError(f"MCP 错误 [{err.get('code')}]：{err.get('message')}")
    result = resp.get("result")
    if not result:
        raise RuntimeError(f"MCP 返回空 result：{name}")
    content = result.get("content") or []
    if not content or "text" not in content[0]:
        raise RuntimeError(f"MCP 返回无文本内容：{name}")
    try:
        return json.loads(content[0]["text"])
    except ValueError:
        return content[0]["text"]
```

**速度优势**：直接 POST `/api/mcp/tools`，跳过 SSE 握手（SSE 有 cold start 延迟）。

---

## 10 个 MCP 工具

> ⚠️ 工具命名按 Meta 三层结构：Campaign → **Ad Set** → Ad（对应 TikTok 的 Campaign → Adgroup → Ad）。`get_adgroups`/`create_adgroup` 在本技能中更名为 `get_adsets`/`create_adset`。

### 1. list_advertisers — 列出所有广告主
```python
advs = mcp("list_advertisers")
# 返回 [{id, advertiserId, advertiserName, account_status, balance, currency, timezone,
#          campaignsCount, adsetsCount, adsCount, creativeCount,
#          lastEntitySyncAt, lastReportSyncAt, ...}]
```
- `balance` 为**字符串或 `null`**（`null` = 未同步，不是 $0）。
- `account_status`: `ACTIVE` / `DISABLED` / `UNSETTLED` / `PENDING_REVIEW` / `IN_GRACE_PERIOD`。
- **巡检/报表起点**：先调它，再按 `campaignsCount > 0` 过滤活跃账户。

### 2. get_alerts — 获取系统预警
```python
alerts = mcp("get_alerts", {"advertiser_id": "act_1234567890", "limit": 20})
```
- 必填: `advertiser_id`；可选: `is_read` (bool), `limit` (默认 20)。

### 3. get_campaigns — 获取 Campaign 列表
```python
camps = mcp("get_campaigns", {
    "advertiser_id": "act_1234567890",
    "status": "ACTIVE",      # 可选: ACTIVE / PAUSED / DELETED / ARCHIVED；传 "all" 不过滤
    "search": "关键词",       # 可选：名称模糊搜索
    "page": 1, "pageSize": 50
})
# 返回 {"data": [...], "total": N, "page": 1, "pageSize": 50}
```
- ⚠️ 过滤值用 `ACTIVE`/`PAUSED`，**不要用 TikTok 的 `ENABLE`/`DISABLE`**（Meta 非有效枚举）。
- 返回是 `{"data":[...]}`，取 `.data`。

### 4. create_campaign — 创建 Campaign
```python
camp = mcp("create_campaign", {
    "advertiser_id": "act_1234567890",
    "campaign_name": "我的新Campaign",
    "objective": "SALES",                  # Meta 官方简化值，见 enums-reference §1
    "daily_budget": "100.00",              # 日预算（字符串）；或 lifetime_budget 设总预算
    "budget_optimization": True,           # CBO / Advantage+ Campaign Budget
    "bid_strategy": "LOWEST_COST_WITHOUT_CAP"  # 见 enums-reference §5
})
# 返回 id=本地UUID；线上ID在 facebookCampaignId
```

### 5. get_adsets — 获取 Ad Set 列表
```python
adsets = mcp("get_adsets", {
    "advertiser_id": "act_1234567890",
    "campaign_id": "本地的 campaign UUID",  # 可选，非 facebookCampaignId
    "status": "ACTIVE",
    "search": "名称搜索", "page": 1, "pageSize": 50
})
```

### 6. create_adset — 创建 Ad Set
```python
adset = mcp("create_adset", {
    "advertiser_id": "act_1234567890",
    "campaign_id": "本地的 campaign UUID",   # 必填
    "adset_name": "广告组名称",               # 必填
    "targeting": {                           # 必填
        "geo_locations": {"countries": ["US"]},  # 必填: 国家
        "genders": [1],                           # 1=全部 2=男 3=女
        "age_min": 18, "age_max": 65,
        "interests": [], "behaviors": [],
        "custom_audiences": [], "lookalike_audiences": [],
        "advantage_audience": True               # Advantage+ 受众扩展
    },
    "daily_budget": "50.00",
    "optimization_goal": "OFFSITE_CONVERSIONS",  # Meta 官方值，见 enums-reference §3
    "billing_event": "IMPRESSIONS",              # Meta 官方值，见 enums-reference §4
    "bid_strategy": "LOWEST_COST_WITHOUT_CAP",   # 见 enums-reference §5
    "placement_type": "automatic",               # automatic / manual（Advantage+ Placements）
    "start_time": "2026-07-13T00:00:00",
    "status": "PAUSED"                           # 建议先 PAUSED，确认后改 ACTIVE
})
```

### 7. get_ads — 获取 Ad 列表
```python
ads = mcp("get_ads", {
    "advertiser_id": "act_1234567890",
    "adset_id": "本地 adset UUID", "status": "ACTIVE",
    "search": "搜索词", "page": 1, "pageSize": 50
})
```

### 8. create_ad — 创建 Ad
```python
ad = mcp("create_ad", {
    "advertiser_id": "act_1234567890",
    "adset_id": "本地 adset UUID",       # 必填
    "ad_text": "广告文案",                # 必填
    "ad_name": "可选名称",
    "creative_id": "素材库UUID",          # 需先用面板/浏览器上传（MCP 暂无上传接口）
    "image_url": "图片URL",
    "video_id": "已上传的视频ID",
    "call_to_action": "SHOP_NOW",         # 见 enums-reference CTA 枚举
    "landing_page_url": "落地页URL",
    "page_id": "Facebook 主页ID",         # Meta 广告需绑定主页
    "instagram_actor_id": "Instagram 账号ID"  # 可选
})
```
- ⚠️ **MCP 暂无素材上传工具**：`creative_id`/`video_id` 需先通过面板或浏览器上传取得。
- ⚠️ **Meta 广告需绑定 Facebook 主页**（`page_id`），无素材或无主页时按 ad-creation-flow 的兜底步骤引导，不要直接提交创建。

### 9. get_daily_metrics — 日报指标
```python
metrics = mcp("get_daily_metrics", {
    "advertiser_id": "act_1234567890",
    "start_date": "2026-07-01", "end_date": "2026-07-07",
    "level": "ACCOUNT"   # ACCOUNT / CAMPAIGN / ADSET / AD
})
```
- 必填: `advertiser_id`, `start_date`, `end_date`；`level` 默认按账户聚合。
- 下钻层级（CAMPAIGN/ADSET/AD）**可能为空**：调用后若为空，降级 ACCOUNT 并在报告中注明，不编造明细（见 enums-reference §10）。
- Meta 关键指标：`spend`、`impressions`、`clicks`、`ctr`、`cpc`、`cpm`、`conversions`、`purchase_roas`、`cost_per_action_type`、`frequency`、`reach`。

### 10. sync_advertiser_data — 同步实体数据
```python
mcp("sync_advertiser_data", {
    "advertiser_id": "act_1234567890",
    "levels": ["ACCOUNT","CAMPAIGN","ADSET","AD"]
}, timeout=60)
```
- ⚠️ 大账户极易 60s+ 超时，**交互式查询默认不调用**；仅用户显式指定账户且缓存陈旧时使用，失败提示去平台手动刷新（见 mcp-client.md §8）。

---

## 业务模块总览

基于 MCP 工具实现三大模块。**触发关键词决定走哪个模块**；多关键词命中时按「创建 → 巡检 → 报表」顺序处理。模块详细规则分别在 `references/` 下。

### 触发关键词（已收窄，降低误触）

| 模块 | 关键词 |
|------|--------|
| 广告创建 | 创建广告、建广告、投广告、投放广告、我要投放、帮我投放、创建Facebook广告、创建FB广告、搭建广告、批量创建广告、批量投放、帮我把广告发布出去、推广投放 |
| AI 巡检 | 广告巡检、巡检、开启巡检、启动巡检、开始巡检、监控广告、AI巡检 |
| 日报/周报 | 生成日报、给我生成日报、生成周报、给我生成周报、日报数据、昨天日报、今日数据报告、周报数据、本周周报、根据我的格式生成日报、根据我的格式生成周报 |

> 说明：「确认 / 好的 / 需要 / 开始」**不是全局触发词**。它们仅在**本技能已发出巡检询问的上下文**中，表示用户同意开启巡检；不会因对话里出现这些词就误触发。

指定日期/账户格式：`x天日报`、`x日-x日的日报`、`广告账户名称 xxx 的日报/周报`、`广告系列名称 xxx 日报/周报` 等。

---

## 巡检自动触发条件（上下文式）

除用户主动说巡检关键词外，以下场景**先询问、不自动执行**；用户在该询问上下文里回「确认/好的/需要/开始」才真正开启：

### 场景 1：广告创建完成后
> "广告已创建完成，是否需要帮您创建广告巡检呢？覆盖花费、转化、CPA、ROAS、预算、素材等方面进行AI监测，可以回复「确认」进行开启"

### 场景 2：预算花费超过 60%
> "当前检测到您的广告预算已花费超 60%，是否需要开启 AI 广告巡检呢？可以回复「确认」进行开启"

### 场景 3：检测到有在投放的广告
> "检测到您有正在投放的广告，是否需要开启 AI 广告巡检呢？可以回复「确认」进行开启"

### 场景 4：每日定时主动询问
- 每天早上 **9:45**、下午 **15:00** 各询问一次（由定时任务触发，见下文）。

---

## 模块一：广告创建（详见 references/ad-creation-flow.md）

投放前检查：账户授权、余额、营销目标、优化目标、素材（已上传+审核通过）、主页、定向、预算、出价。不通过不得进入提交，并提示原因。

创建流程按面板依次收集：账户 → 营销目标 → 优化目标 → 素材 → 主页 → 定向 → 预算 → 出价。预算检测：同账户/目标/优化下，对比 7 日均值，超阈值提示建议区间。所有高风险操作（创建/修改/发布）须用户明确确认后方可执行。

**MCP 工具映射**：`list_advertisers` → `create_campaign` → `create_adset` → `create_ad`。

创建完成返回状态：成功（返回各层 ID + 触发巡检询问）/ 失败（重试≤3 → 仍失败给原因+建议字段）/ 进行中（明确提示"等待接口返回"，不默认成功）。**输出不得编造任何结果或状态**。

---

## 模块二：AI 广告巡检（详见 references/ad-inspection-rules.md）

提醒广告数据变化趋势。覆盖六类异常（按优先级）：花费异常 > 转化异常 > CPA 异常 > ROAS 异常 > 预算耗尽 > 素材疲劳。

**MCP 执行（两步滤波法，全程 < 10s，见 mcp-client.md §4）**：
1. `list_advertisers` → 过滤 `campaignsCount > 0`
2. 并行 `get_daily_metrics(level=ACCOUNT)` → 找今日真有数据的账户
3. 只对有数据的账户拉 7 天历史做阈值对比

> ⚠️ 不调 `sync_advertiser_data`（交互场景）⚠️ `as_completed` 设 `timeout=30` ⚠️ 单次 `timeout=15`

---

## 模块三：日报/周报（详见 references/report-format.md）

生成日报/周报，支持原模板或自定义格式。日报重今日表现/环比/问题广告/优秀广告/素材亮点/风险/明日动作；周报重趋势/归因/复盘/复制方向/下周动作。

**MCP 执行**：
1. `list_advertisers`
2. 并行 `get_daily_metrics`（advertiser_id + 日期区间）
3. 尝试 `level=CAMPAIGN/ADSET` 下钻识别问题/优秀广告；**若为空降级 ACCOUNT 并注明**，不编造
4. 按 report-format 输出 Markdown

账户过滤：仅已授权且 `balance` 非 `null` 的账户；指定账户时按 mcp-client §8 处理（不盲目 sync）。

---

## 输出渲染策略（降级）

1. 支持 ui_payload → 结构化 JSON + ui_payload（巡检概览/异常卡/建议；日报指标卡/环比/广告列表）
2. 不支持 → Markdown 卡片
3. 仍不行 → 简洁摘要，**禁止**在界面直接展示完整 raw JSON

---

## 定时任务

- 定时巡检：「每天x点开启巡检」→ 创建定时任务到点执行并推送；「关闭定时巡检」→ 移除。
- 定时日报/周报：「每天x点生成日报」「每周星期x生成周报」→ 创建定时任务；「根据我的格式生成日报」→ 保存为模板。
- 定时能力由宿主平台提供（cron 表达式 + 宿主时区）；若平台无定时能力，改为在会话内提示用户手动触发。

---

## 报告生成成功判定

全部满足才算成功：1) 已识别关键词；2) 已读取广告主列表；3) 已过滤有效账户；4) 已获取指标；5) 已生成渲染/Markdown；6) 回复优先展示在页面。缺数据不编造。

---

## ⚠️ 常见坑（统一口径）

1. **`sync_advertiser_data` 极慢易超时**：交互场景默认不调，让用户平台刷新（mcp-client §8）。
2. **下钻 level 可能为空**：优先 ACCOUNT；CAMPAIGN/ADSET/AD 为空则降级并注明，不编造。
3. **两步滤波法**替代全量扫：先 `campaignsCount>0` 过滤，再查 ACCOUNT 找今日有数据账户，只对这些深度分析（<10s vs 全量 5min）。
4. **`ThreadPoolExecutor.as_completed` 必须 `timeout=30`**；单次 `timeout=15`。
5. **`get_*` 返回 `{"data":[...]}`**，取 `.data`。
6. **`create_*` 返回本地 UUID**，线上 ID 在 `facebookXxxId`；后续 `get_*` 用本地 UUID。
7. **数值字段是字符串**：`balance`/`spend`/`amount_spent` 等需 `float()` 后格式化（用 `fmt_bal`）。
8. **`status` 用 `ACTIVE`/`PAUSED`，不用 TikTok 的 `ENABLE`/`DISABLE`**（Meta 非有效枚举）。
9. **`balance: null`** = 未同步，不是 $0。
10. **`DISABLED` / `IN_GRACE_PERIOD` 账户** Campaign 查询可能返回空。
11. **大账户分页**：`pageSize` 最大 200。
12. **枚举使用 Meta 官方值**：`objective`/`optimization_goal`/`billing_event`/`bid_strategy` 均用 Meta Marketing API 官方枚举（如 `SALES`、`OFFSITE_CONVERSIONS`、`IMPRESSIONS`、`LOWEST_COST_WITHOUT_CAP`），完整对照与可直接导入的 Python 常量见 `references/enums-reference.md`；调用前用 `validate_args` 预校验（mcp-client §7，已同时放行旧版官方值，确保不报错）。**VidAU 若对字段做简化封装，需以真实接受值为准校准后更新**。
13. **层级名用 Ad Set**：工具名为 `get_adsets`/`create_adset`，不是 TikTok 的 `adgroup`。
14. **Meta 广告需绑定 Facebook 主页**（`page_id`），创建 Ad 前确认主页已绑定。
15. **平台域名待校准**：默认 `facebook.vidau.ai`，真实域名以 VidAU 部署为准（见 mcp-client §1）。

---

## 打开面板 / 浏览器 fallback（明确边界）

仅以下 MCP 做不到的场景才用浏览器：
- 「打开面板/打开 Facebook 后台」→ 用 `subprocess` 启本地已登录 Chrome（不走 browser MCP，Meta 会拦自动化浏览器）。
- 素材上传、Meta OAuth、主页管理 → fallback 浏览器/面板操作。

```python
import subprocess, os
chrome_paths = [
    r"C:\Program Files\Google\Chrome\Application\chrome.exe",
    r"C:\Program Files (x86)\Google\Chrome\Application\chrome.exe",
    os.path.expandvars(r"%LOCALAPPDATA%\Google\Chrome\Application\chrome.exe"),
]
chrome = next((p for p in chrome_paths if os.path.exists(p)), None)
if chrome:
    subprocess.Popen([chrome, "--new-window", "https://facebook.vidau.ai/"],
                     stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
```

可用脚本：`~/AppData/Roaming/vidau/scripts/open_facebook_panel.py`

---

## 与 ads-tiktok-mcp 的区别（同构对照）

| | ads-facebook-mcp | ads-tiktok-mcp |
|---|---|---|
| 媒体渠道 | Facebook / Meta | TikTok |
| 三层结构 | Campaign → **Ad Set** → Ad | Campaign → Adgroup → Ad |
| 状态枚举 | `ACTIVE` / `PAUSED` | `ENABLE` / `DISABLE` |
| 营销目标 | `AWARENESS`/`TRAFFIC`/`ENGAGEMENT`/`LEADS`/`APP_PROMOTION`/`SALES` | `REACH`/`TRAFFIC`/`VIDEO_VIEWS`/`LEAD_GEN`/`WEB_CONVERSIONS` 等 |
| 出价策略 | `bid_strategy`（LOWEST_COST_WITHOUT_CAP / COST_CAP 等） | `bid_type`（BID_TYPE_NO_BID / BID_TYPE_CUSTOM） |
| 版位体系 | `publisher_platforms`（facebook/instagram/audience_network/messenger） | `placement_type`（TIKTOK/PANGLE 等） |
| 转化追踪 | Meta Pixel + Conversions API + Advanced Matching | TikTok Pixel + Events API |
| 自动优化 | Advantage+ 系列（Campaign Budget / Audience / Creative / Shopping） | CBO / Maximum Delivery |
| 学习期 | 50 次转化 / 7 天 | 50 次转化 / 7 天 |
| 平台域名 | `facebook.vidau.ai`（待校准） | `tiktok.vidau.ai` |
| 鉴权 | 环境变量 `FACEBOOK_MCP_API_KEY` | 环境变量 `TIKTOK_MCP_API_KEY` |
| 广告主页 | 需绑定 Facebook 主页（page_id） | 不需要 |
| 创意主比例 | Feed 4:5 / Stories-Reels 9:16 | 9:16 为主 |
| 方式 | HTTP JSON-RPC 直连 VidAU MCP | 同 |
| 适用 | 巡检、查询、轻量写入、广告创建 | 同 |

**规则**：能用 MCP 就用 MCP，只有它搞不定的（上传素材、Meta OAuth、开面板）才 fallback 浏览器。

---

## 版本基线与变更（Changelog）

- **对齐基线**：基于 **Meta Marketing API 官方枚举** + 与 `ads-tiktok-mcp v2.3.0` 同构设计；VidAU 封装层若做简化需以真实 MCP 接受值为准校准。
- **v2.3.0**（本次，首版 Facebook bundle）
  - 按 Meta Marketing API 官方枚举编写：`objective`（AWARENESS/TRAFFIC/ENGAGEMENT/LEADS/APP_PROMOTION/SALES）、`optimization_goal`、`billing_event`、`bid_strategy`（LOWEST_COST_WITHOUT_CAP 等）、`publisher_platforms`。
  - 适配 Meta 三层结构：Ad Set 替代 Adgroup（`get_adsets`/`create_adset`）。
  - 状态枚举切为 `ACTIVE`/`PAUSED`（非 TikTok 的 `ENABLE`/`DISABLE`）。
  - 巡检规则六类异常保留，排查/止损/优化建议全面适配 Meta 特性（Advantage+、学习期、Pixel+CAPI、Advanced Matching、Meta 版位等）。
  - 素材规格按 Meta 各版位（Feed 4:5、Stories/Reels 9:16）。
  - 平台域名 `facebook.vidau.ai` 作为同构假设，标注待真实部署校准。
  - 保留 TikTok 版全部工程化能力：API Key 外置、mcp() 容错、重试退避、两步滤波、时区、字段类型、健康检查、巡检误触收窄、status 统一、level 降级、sync 策略统一。

> 升级提示：本版本以 Meta 官方枚举为准，部署后请用 `mcp_healthcheck()` 验证连通性，并按真实 VidAU Facebook MCP 接受值校准枚举（若 VidAU 做了简化封装）。
