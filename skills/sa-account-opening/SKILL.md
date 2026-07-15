---
name: sa-account-opening
description: 开户管理 Skill（账户维度闭环）。当用户表达"开户/开账户/新建账户/开广告户/创建账户/我要开户/帮我开户/开FB户/开Facebook户/开TikTok户/开谷歌户/开Kwai户"等开户意图时触发。必须先读取【当前登录账户】已配置的开户类型（按媒体渠道分组展示供选择）；若无权限或未配置任何开户类型，则阻断并引导找运营配置开户政策；确认类型后再按平台收集字段并提交。适用于 VIDAU AGENT 端口，遵循"关键词触发 → API 调用 → 端口回复 → 面板调起"。
agent_created: true
---

# 开户管理 Skill（账户维度闭环）

## 概述

本 Skill 解决的核心问题：**开户不能不问类型就执行，也不能把全局列表甩给用户自己挑**。

正确流程是账户维度的：

1. 先确认**当前登录账户是谁、有没有开户权限、配了哪些媒体渠道**（`/user/getInfo`）。
2. 读取**该账户已配置的开户类型**（由账户 api-key 限定的 `busList`），按媒体渠道分组，**以列表形式展示给用户挑选**。
3. 若账户**无开户权限 / 未配置任何开户类型** → 立即阻断，调出**运营联系人**引导其先配置开户政策，绝不臆造类型继续。
4. 用户选定类型后，再按平台收集字段 → 展示 payload → 等用户确认"提交" → 调提交接口。

所有数据来源必须是接口返回，绝不编造；所有选项来自"当前登录账户"，不把全局字典当作用户可选项直接甩出。

## 触发词

开户、开账户、新建账户、开广告户、创建账户、我要开户、帮我开户、开 FB 户、开 Facebook 户、开 TikTok 户、开谷歌户、开 Kwai 户、开 Bigo 户、创建广告账户、申请开户

## VIDAU 端口对接规则（必须执行）

1. 触发关键词后，所有询问、列表展示、payload 预览、回执都必须在 **VIDAU AGENT 端口**回复展示。
2. 读取与展示均通过 `customerapi_call`（广告投放 MCP）完成；优先用接口数据，不依赖浏览器点击表单。
3. 高风险操作（提交开户申请）须等用户明确回复"提交"后方可执行。
4. 不编造任何开户类型、字段选项或提交结果；接口未返回则明确提示当前状态。
5. 支持 `ui_payload` 渲染（见末节「端口渲染字段」）；不支持时降级为 Markdown 卡片，禁止直接输出 raw JSON。

## 核心铁律

1. **第一铁律（先鉴权+列类型，绝不跳过）**：收到开户意图后，必须先完成 Phase 0 + Phase 1（账户鉴权、读取该账户已配置开户类型并分组展示）。**在用户选定 `AdsBusType` 之前，不收集任何字段、不调任何提交接口**。这从根本上杜绝"不问类型就执行"。
2. **账户维度优先**：所有可选项必须来自"当前登录账户"的已配置数据。不得把全局 `busList` 当作用户可选项直接甩出——需按账户 api-key 读取、按平台分组、去噪后再展示。
3. **无权限/未配置即阻断**：若账户无「开户申请」菜单权限，或已配置开户类型为空，立即阻断并引导找运营，绝不臆造类型继续往下走。

---

## 执行流程

### Phase 0 — 账户鉴权与权限自检（每次开户必跑）

调用：
```
customerapi_call(path="/user/getInfo", method="POST", body={})
```

从返回 `data` 中读取并暂存：
- `clientName`（账户名）、`clientId`、`userName`
- `menus` 数组：是否存在 `Title == "开户申请"` → 有则**有权限**，无则**无权限**
- `mediaAgentIds`：该账户已配置的媒体代理渠道（如 `[11,13,18,41,...]`）

**分支判定：**

- ❌ `menus` 中无「开户申请」**或** `mediaAgentIds` 为空 → 跳转 **Phase 0b（找运营）**。
- ✅ 有权限且已配渠道 → 进入 **Phase 1**。

#### Phase 0b — 找运营（无权限 / 未配置时阻断输出）

先取真实运营联系人：
```
customerapi_call(path="/client/getCustomerService", method="POST", body={"clientId": <clientId>})
```
返回含 `customerServiceNameDomestic`（国内客服）、`customerServiceNameAbroad`（海外运营）、`email`、`customerServiceImg`（二维码）。

在端口输出（ui_payload 或 Markdown 卡片）：
> ⚠️ 您的账户「<clientName>」当前**未配置开户权限 / 未配置开户类型**，无法在此直接开户。
> 请先联系运营人员为您配置**开户政策与开户表单**：
> - 国内客服：<customerServiceNameDomestic>
> - 海外运营：<customerServiceNameAbroad>
> - 邮箱：<email>
> 配置完成后，回来告诉我"我要开户"即可继续。

并输出 `blocked`，原因 `account_opening_not_configured`。**流程结束，不继续收集字段。**

---

### Phase 1 — 读取该账户已配置开户类型（按媒体渠道分组展示）

调用：
```
customerapi_call(path="/adAccountApply/busList", method="POST", body={})
```

说明：该列表由当前账户 `api-key` 限定（**生产环境仅返回本账户已配置类型**；dev 测试 key 会返回全部，需走下方去噪规则）。`busList` **不认 `adsType` 入参**，必须客户端按返回项的 `AdsType` 字段分组。

**AdsType → 平台映射**（由 busList 全量枚举得出，共 18 种媒体渠道；提交端点仅 6 类：`google`/`facebook`/`tiktok`/`kwai`/`bigo`/`add_otherMedia`）：

| AdsType | 平台 | 提交端点 | 字段来源 |
|---|---|---|---|
| 1 | Google | `/adAccountApply/google` | 文档 2.3（详） |
| 2 | Facebook | `/adAccountApply/facebook` | 文档 2.1（详） |
| 3 | TikTok | `/adAccountApply/tiktok` | 文档 2.2（详） |
| 4 | Twitter | `/adAccountApply/add_otherMedia` | 其他媒体（运营配置） |
| 5 | Line | `/adAccountApply/add_otherMedia` | 其他媒体（运营配置） |
| 6 | Bing | `/adAccountApply/add_otherMedia` | 其他媒体（运营配置） |
| 7 | Unityads | `/adAccountApply/add_otherMedia` | 其他媒体（运营配置） |
| 8 | Kwai | `/adAccountApply/kwai` | 文档 2.4（详） |
| 9 | 传音 Transsnet | `/adAccountApply/add_otherMedia` | 其他媒体（运营配置） |
| 10 | OPPO | `/adAccountApply/add_otherMedia` | 其他媒体（运营配置） |
| 11 | 华为 Huawei | `/adAccountApply/add_otherMedia` | 其他媒体（运营配置） |
| 12 | Bigo | `/adAccountApply/bigo` | Bigo 表单（运营配置） |
| 16 | Snapchat | `/adAccountApply/add_otherMedia` | 其他媒体（运营配置） |
| 17 | Yandex | `/adAccountApply/add_otherMedia` | 其他媒体（运营配置） |
| 18 | Taboola | `/adAccountApply/add_otherMedia` | 其他媒体（运营配置） |
| 19 | VKontakte(VK) | `/adAccountApply/add_otherMedia` | 其他媒体（运营配置） |
| 21 | Outbrain | `/adAccountApply/add_otherMedia` | 其他媒体（运营配置） |
| 22 | Xandr | `/adAccountApply/add_otherMedia` | 其他媒体（运营配置） |

> 说明：busList 在 dev key 下返回全部 201 条（跨 18 个 AdsType）；**生产环境由账户 api-key 限定，仅返回本账户已配置的类型**，所以 Phase 1 实际展示的渠道与类型以实时返回为准。本表是"所有可能渠道"的映射，不是每次都全展示。

**去噪规则（dev 环境混入大量测试项，生产环境可跳过）：**
排除 `AdsBusName` 命中以下任一模式（先 `strip()` 去首尾空格、按全角/半角归一后再判）：
- 正则 `/^(测试|test|dev|\d+)$/i` 或纯数字
- **测试前缀**：`^Test` / `^ttww`（任意大小写）→ 命中 `Test12gg`/`Test-2`/`ttww2~8` 等
- **各平台测试克隆**（`*C1.1` 系列）：`GC1.1`/`F1.1`/`TC1.1`/`BC1.1`/`UC1.1`/`KC1.1`/`SC1.1`/`OC1.1`/`HC1.1` → 正则 `/^\w*C1\.1$/i`
- **平台名+数字尾巴克隆**：正则 `/^(tiktok|tt|tik tok|gg|fb|google|kwai|bing|line|huawei|oppo|bigo|snapchat|yandex|taboola|vk|xandr|unityads|outbrain)\d+$/i` → 命中 `tiktok111`/`TT1`/`tiktok222`/`tiktok33` 等
- 重复项去重：同名（归一后）仅保留首条（如 `Twitter`/`Kwai`/`Line`/`FB`/`gg`/`TT` 各出现多次，重复名视为噪音克隆）
- 明显乱码/无意义：`的方法大姑`、`很好，非常好`、`666`、`QWQR`、`DNMD-GG`、`黑悟空大媒体`、`taest214`、`wwwwwww三不限`、`华哥6661`、`https_Google`、`赛道--TT测试`、`Test-1/2/3`、`TestOutbrain`、`line1`、`Coke`、`Google-1`

保留其余业务项（如 `FB国内户`、`TikTok(美国小店2)`、`GG南美户`、`Twitter`、`Kwai-GBP`、`传音`、`OPPO`、`华为`、`Bigo` 等）。

**在端口展示分组列表（ui_payload `opening_types` 或结构化表格组件）—— 严格按媒体渠道分组，禁止平铺：**

输出格式**必须**为「每个平台一个分组标题 + 组内以**表格（三列：类型名称 / 类型码 / 备注）或结构化列表项**逐条列出」，**绝不能用 `·` 或 `,` 把所有类型拼成一长行文本**：

> 正确渲染方式（结构化表格，每项独立成行；**⚠️ BC ID 列仅在 TikTok 渠道出现，其余所有渠道一律不显示此列**）：
>
> ┌──────────────────────┬──────┬──────────┐
> │ 【Facebook】          │      │          │
> ├──────────────────────┼──────┼──────────┤
> │ FB国内户             │  2   │          │
> │ FB南美户(AC)         │  5   │ 即冲即返 │
> │ FB尼日户            │  29   │          │
> │ FB-GBP              │ 115   │ 英镑     │
> │ …                    │      │          │
> └──────────────────────┴──────┴──────────┘
>
> ┌──────────────────────┬──────┬──────────┬────────┐
> │ 【TikTok】            │      │          │        │
> ├──────────────────────┼──────┼──────────┼────────┤
> │ TikTok(非小店)       │  4   │          │ 必填   │
> │ TT南美户            │ 104   │          │ 必填   │
> │ …                    │      │          │        │
> └──────────────────────┴──────┴──────────┴────────┘
>
> **注意：TikTok 表格有四列（类型名称/类型码/备注/BC ID），其余渠道仅三列（类型名称/类型码/备注），绝不给非 TikTok 渲染 BC ID 列。**

若环境支持交互组件（Visualizer / HTML widget），优先使用**可筛选的分组表格**（含「全部/国内户/海外户/代理商」筛选按钮），每项独立一行、三列对齐。

⚠️ **渲染铁律（违反即视为 Skill 执行错误）：**
1. **禁止平铺** —— 禁止将开户类型用 `·` `,` `/` 等符号拼接成没有【平台】分组标题的一长行文字（反面：「FB国内户（2）· FB南美户(AC)（5）· FB尼日户（29）…」）。
2. **必须按 AdsType 平台分组** —— 每组带【平台】标题，用户才能清楚每个类型归属哪个渠道、该走哪个提交端点。
3. **每项必须独立成行** —— 表格或列表形式，包含至少「类型名称 + 类型码」两列，可选第三列「备注」。
4. **BC ID 列仅限 TikTok (AdsType=3)** —— 只有 TikTok 渠道的表格才展示 BC ID 列（四列），**其余 17 个渠道（Facebook/Google/Kwai/Bigo/Twitter/Line/Bing/其他媒体等）一律三列（类型名称/类型码/备注），绝不出现「BC ID: 无」或任何形式的 BC ID 列**。

用户选定 → 记录 `AdsBusType` + 对应 `AdsType`（平台）+ 平台提交端点。

> 若用户想要的类型不在列表中 → 提示"该类型未对您的账户配置，请联系运营开通"，回到 Phase 0b 联系人，不擅自选用其它类型。

---

### Phase 2 — 按开户类型(ADSBUSTYPE)动态渲染表单（用户选定类型后）

⚠️ **核心：表单字段由"开户类型(AdsBusType)"决定，不是只按平台(AdsType)一刀切，更不是只覆盖几个大平台。** 本 Skill 已对 busList 枚举出的 **全部 18 种媒体渠道**（Facebook/TikTok/Google/Kwai/Bigo + Twitter/Line/Bing/Unityads/传音/OPPO/华为/Snapchat/Yandex/Taboola/VK/Outbrain/Xandr）实现动态表单。同一平台下，国内户/海外户的字段不同（如 TikTok 海外户必填 BC ID、国内户不展示 BC ID）；⚠️ 但 **BC ID 仅 TikTok 渠道存在**，Facebook / Google / Kwai / Bigo 及所有其他媒体渠道「均没有」BC ID，海外户也绝不因此追加 BC ID。选定类型后，必须先判定 segment，再渲染对应变体表单。

**2.0 判定 segment（从 `AdsBusName` 语义判定，文档 2.1~2.4 为依据）**
- `isOverseas`（海外户）：命中海外语义词——`海外户 / 南美户 / 尼日户 / GBP / 即冲即返 / 全球即冲即返 / 泰铢 / 越南盾 / 英镑 / 欧洲户 / 北美户 / 埃及户 / ARS / NGN / EGP / JPY / CNY`
- `isDomestic`（国内户）：命中 `国内户 / 非小店 / 二不限 / 现返`
- `ambiguous`（歧义，如含「小店」）：默认按海外渲染（首充金额类型页内先填 + 钱包校验；若属 TikTok 渠道则另含 BC ID★）。如为国内小店请告知我；Facebook 渠道无 BC ID，无需处理
- 币种联动（仅 Google）：`国内户→USD`、`尼日户→NGN`、`AC→投放类型APP`、`NAC→投放类型电商`

**2.1 平台基础字段（按 `AdsType` 取，见文档真实字段）+ 拉真实字典**
- 开户主体：`/company/list`（选已有或 `/company/add` 新建；主体名须与营业执照一致）
- 行业类型：`/data/industrys`
- 代理渠道：`/media/getMediaList`(platformId) 或账户 `mediaAgentIds`

| 平台（AdsType） | 基础字段（采集来源：先拉真实字典） |
|---|---|
| **Facebook (2)** | 首充金额类型(相同金额/不同金额)、开户数量、产品类型(游戏/应用/电商/品牌/APP)、FB主页、推广网站、媒体代理(`/media/getMediaList` platformId=2，选后出 OE 链接)、OE编号、账户时区(**下户后不可改**) —— 文档 2.1 |
| **TikTok (3)** | 开户数量、推广链接、广告时区(**下户后不可改**) —— 文档 2.2 |
| **Google (1)** | 开户数量、默认国家/地区(CID)、默认时区、推广链接、币种(联动)、投放类型(联动) —— 文档 2.3 |
| **Kwai (8)** | 官方网站/产品链接、产品或品牌名称、投放内容所属行业、投放流量地国家、开户数量、推广链接 —— 文档 2.4 |
| **Bigo (12)** | Bigo 专属表单字段（运营配置；无公开文档则按通用基线 + 运营口径） |
| **其他媒体 (4/5/6/7/9/10/11/16/17/18/19/21/22)** | **通用基线**：开户主体(`/company/list`)、开户数量、推广链接、行业类型(`/data/industrys`)。其余专属字段由运营在账户后台配置的开户表单决定（文档 2.5："联系专属客服配置开户政策，再根据开户页面补充"）。**若该账户未配置此渠道表单 → 走 Phase 0b 找运营**，不臆造字段。 |

> 所有平台通用前置：开户主体 `/company/list`、行业 `/data/industrys`、代理渠道 `/media/getMediaList`(platformId) 或账户 `mediaAgentIds`。

**2.2 按 segment / 平台叠加条件字段**

⚠️ **核心口径（重要修正）：BC ID 是 TikTok 渠道专属字段，仅 TikTok(3) 存在。Facebook(2) / Google(1) / Kwai(8) / Bigo(12) 及所有其他媒体渠道「均没有」BC ID，海外户也绝不因此追加 BC ID。** 海外/国内差异体现在首充金额类型、国家/地区、钱包校验等字段上，这些仍按 segment 生效，但与 BC ID 无关。判定 `isDomestic`/`isOverseas` 见 2.0。

| 条件 | 追加字段 | 隐藏字段 |
|---|---|---|
| **TikTok (3) + `isOverseas`** | **BC ID★**（必填，多个回车隔开）、国家预设(`/tiktok/country/preset/list`)、时区(`/tiktok/common/timezones`)、首充金额类型(页内先填 + 提交前 `/finance/myAccount` 校验钱包) | — |
| **TikTok (3) + `isDomestic`** | 时区★、首充金额类型 | **BC ID 隐藏**（如 TikTok 非小店「非小店」） |
| **任意渠道(非 TikTok) + `isOverseas`** | 国家/地区、首充金额类型(页内先填 + 提交前 `/finance/myAccount` 校验钱包余额 ≥ 首充总额) | BC ID（本就不存在） |
| **任意渠道 + `isDomestic`** | 首充金额类型 | BC ID（本就不存在） |

**平台覆盖项（在 segment 之上叠加）：**

| 平台 | 特殊叠加 / 例外 |
|---|---|
| Facebook (2) | 基础字段见 2.1（开户数量/产品类型/FB主页/推广网站/媒体代理+OE链接/OE编号/账户时区）；**无 BC ID**；海外户首充金额类型页内先填 + 钱包校验 |
| Google (1) | 币种联动(国内户→USD / 尼日户→NGN / 其他见 2.0)、投放类型联动(AC→APP / NAC→电商)；无 BC ID |
| Kwai (8) | 基础字段见 2.1（官网/产品链接/产品品牌名/行业/流量地国家/数量/推广链接）；**无 BC ID**（注：此前误写为"BC ID 始终展示"，已修正为 TikTok 专属） |
| Bigo (12) | Bigo 专属表单（运营配置）；无 BC ID |
| 其他媒体(4/5/6/7/9/10/11/16/17/18/19/21/22) | 通用基线(主体/数量/推广链接/行业) + 运营配置专属字段；**无 BC ID**；海外户首充金额类型页内先填 + 钱包校验 |

**2.3 渲染该变体表单（只展示适用字段，隐藏项绝不列出）**
在端口展示 `collect_form`（`ui_payload.collect_form`），字段带 `required` 标记；明确告知用户"以下为该开户类型专属表单"。

收集完毕后，**展示完整 JSON payload 预览**（`ui_payload.submit_payload`），并提示：
> 请确认以上信息，回复"提交"即生成开户申请单（需运营审核）。

**未收到用户"提交"确认前，禁止调用提交接口。**

---

### Phase 3 — 提交

用户确认后：
```
customerapi_call(path="<平台提交端点>", method="POST", body=<payload>)
```
提交成功后返回申请单号 → 在端口回执（`ui_payload.result`）：
> ✅ 已为账户「<clientName>」提交 <开户类型名> 开户申请，申请单号：<sn>，状态：待运营审核。

---

## 开户类型 → 表单变体 速查（串联示例）

选定类型后，**Phase1→Phase2 串联**渲染的字段集必须如下（隐藏项一律不展示）：

| 选定开户类型(示例) | AdsType | segment | 提交端点 | 渲染出的字段（必填★） |
|---|---|---|---|---|
| TikTok(非小店) [4] | 3 | domestic | `/adAccountApply/tiktok` | 主体★、数量★、推广链接★、时区★、首充金额类型★（**BC ID 隐藏**） |
| TT南美户 [104] | 3 | overseas | `/adAccountApply/tiktok` | 主体★、数量★、推广链接★、时区★、首充金额类型★、**BC ID★**、国家预设★、首充金额(页内填+钱包校验) |
| TikTok-Thb泰铢 [221] | 3 | overseas | `/adAccountApply/tiktok` | 同海外户（BC ID★ + 国家预设★ + 钱包校验） |
| FB国内户 [2] | 2 | domestic | `/adAccountApply/facebook` | 主体★、数量★、产品类型★、FB主页★、推广网站★、媒体代理★(出OE链接)、OE编号★、时区★、首充金额类型★（**无 BC ID；首充不需页内先填**） |
| FB国内户(现返) [105] | 2 | domestic | `/adAccountApply/facebook` | 同 FB国内户；现返政策以运营口径为准 |
| Facebook国内二不限户 [109] | 2 | domestic | `/adAccountApply/facebook` | 同 FB国内户 |
| GG国内户(AC) [1] | 1 | domestic | `/adAccountApply/google` | 主体★、数量★、默认国家(CID)★、时区★、推广链接★、币种=**USD**(联动)、投放类型=**APP**(联动)、首充金额类型★ |
| GG尼日户 [26] | 1 | overseas | `/adAccountApply/google` | 同 GG国内户 但 币种=**NGN**(联动)、首充金额(页内填+钱包校验) |
| Kwai [14] | 8 | — | `/adAccountApply/kwai` | 主体★、官网/产品链接★、产品/品牌名★、行业★、流量地国家★、数量★、推广链接★（**无 BC ID**，BC ID 为 TikTok 专属） |
| Bigo 测试 [106] | 12 | — | `/adAccountApply/bigo` | 按 Bigo 表单字段 |

**其他媒体（长尾渠道，AdsType 4/5/6/7/9/10/11/16/17/18/19/21/22）串联示例** —— 通用基线 + segment 生效，但**均无 BC ID**（仅 TikTok 有）：

| 选定开户类型(示例) | AdsType | segment | 提交端点 | 渲染出的字段（必填★，通用基线 + segment） |
|---|---|---|---|---|
| Twitter(海外) [7] | 4 | overseas | `/adAccountApply/add_otherMedia` | 主体★、数量★、推广链接★、行业★、国家/地区★、首充金额(页内填+钱包校验)（无 BC ID） |
| Line(国内) [32] | 5 | domestic | `/adAccountApply/add_otherMedia` | 主体★、数量★、推广链接★、行业★（无 BC ID） |
| Bing(海外) [11] | 6 | overseas | `/adAccountApply/add_otherMedia` | 主体★、数量★、推广链接★、行业★、国家/地区★、首充金额(页内填+钱包校验)（无 BC ID） |
| 传音(国内) [18] | 9 | domestic | `/adAccountApply/add_otherMedia` | 主体★、数量★、推广链接★、行业★（无 BC ID） |
| OPPO(海外) [20] | 10 | overseas | `/adAccountApply/add_otherMedia` | 主体★、数量★、推广链接★、行业★、国家/地区★、首充金额(页内填+钱包校验)（无 BC ID） |
| 华为(国内) [21] | 11 | domestic | `/adAccountApply/add_otherMedia` | 主体★、数量★、推广链接★、行业★（无 BC ID） |
| Snapchat(海外) [132] | 16 | overseas | `/adAccountApply/add_otherMedia` | 主体★、数量★、推广链接★、行业★、国家/地区★、首充金额(页内填+钱包校验)（无 BC ID） |
| Yandex(国内) [129] | 17 | domestic | `/adAccountApply/add_otherMedia` | 主体★、数量★、推广链接★、行业★（无 BC ID） |
| Taboola(海外) [131] | 18 | overseas | `/adAccountApply/add_otherMedia` | 主体★、数量★、推广链接★、行业★、国家/地区★、首充金额(页内填+钱包校验)（无 BC ID） |
| VK(国内) [130] | 19 | domestic | `/adAccountApply/add_otherMedia` | 主体★、数量★、推广链接★、行业★（无 BC ID） |
| Outbrain(海外) [219] | 21 | overseas | `/adAccountApply/add_otherMedia` | 主体★、数量★、推广链接★、行业★、国家/地区★、首充金额(页内填+钱包校验)（无 BC ID） |
| Xandr(海外) [190] | 22 | overseas | `/adAccountApply/add_otherMedia` | 主体★、数量★、推广链接★、行业★、国家/地区★、首充金额(页内填+钱包校验)（无 BC ID） |

> 注：长尾渠道的"通用基线"为「主体/数量/推广链接/行业」四类；**BC ID 仅 TikTok 渠道存在，其余渠道均无**。海外户的首充金额类型需页内先填并 `/finance/myAccount` 校验钱包余额，国内户仅填类型。各渠道若已被运营在后台配置专属开户表单，则在其上叠加运营字段，且**绝不臆造**文档未列出的字段名。

> 注：`ambiguous`（如含「小店」）默认按海外渲染（首充页内填 + 钱包校验；属 TikTok 渠道时另含 BC ID★）；如运营另有口径，以运营为准。

## 端口渲染字段（ui_payload）

| 字段 | 说明 |
|---|---|
| `account_info` | clientName / clientId |
| `opening_types` | **分组列表（按平台分组，绝不平铺）**：`[{platform: "Facebook", items:[{name, adsBusType}]}, {platform: "TikTok", items:[...]}, ...]`，每组对应一个【平台】标题 |
| `selected_type` | 用户选定项（name + adsBusType + adsType + segment: domestic/overseas） |
| `form_variant` | 该类型渲染出的字段集（已按 segment 增删，含 required 标记） |
| `collect_form` | 待填字段表（字段/必填/可选项） |
| `submit_payload` | 提交前预览 JSON |
| `result` | 提交回执（单号/状态） |
| `blocked` | 阻断时：`{reason: "account_opening_not_configured", contact:{...}}` |

若端口不支持 `ui_payload` → 全部降级为 Markdown 卡片，禁止直接输出 raw JSON。

## 边界与禁忌

- ❌ 禁止跳过 Phase 0/1 直接问字段。
- ❌ 禁止把未去噪的全局 `busList` 直接展示。
- ❌ 禁止未确认即提交。
- ❌ 禁止编造开户类型 / 字段选项 / 提交结果。
- ❌ 禁止在账户无权限时臆造一个类型继续流程。
- ❌ **禁止将开户类型平铺为无【平台】分组标题的单一清单**（见 Phase 1 渲染铁律）。
