# 广告投放 MCP 工具映射表（已连上 · 真实接口）

> 状态：**已连上**（重启 Agent 平台 后 `mcp__{{MCP_CONNECTOR}}__*` 工具已暴露，实测 `/company/list` 返回真实数据）。
> 关键认知：这不是"动作参数"式工具，而是 **path 网关**——每个工具接受一个 `path` + `method` + `body/query`，由网关路由到对应后端 API。

## 四个真实工具

| 工具 | 用途 | 调用形态 | body 命名风格 |
|------|------|----------|--------------|
| `mcp__{{MCP_CONNECTOR}}__customerapi_call` | 广告账户、报表、财务、开户、工单 | `customerapi_call(path="/...", method="POST", body={...})` | **camelCase**（pageNum/pageSize） |
| `mcp__{{MCP_CONNECTOR}}__launchapi_call` | 批量创编、投放查询、同步、状态变更 | `launchapi_call(path="/...", method="GET|POST|PUT", query={...}, body={...})` | 查询用 page/pageSize |
| `mcp__{{MCP_CONNECTOR}}__creativeapi_call` | 素材库（/client/**） | `creativeapi_call(path="/client/...", method, query/body)` | **snake_case**（folder_id/page_size） |
| `mcp__{{MCP_CONNECTOR}}__upload_material` | 上传素材（预签名直传或 by-url） | `upload_material(file_url|file_base64, file_name, ...)` | — |

- catalog 端点数：customerapi 116、launchapi 181、creativeapi 50。
- 调用前可读 MCP Resource 获取完整路径与示例：
  - `{{MCP_CONNECTOR}}://catalog/customerapi`
  - `{{MCP_CONNECTOR}}://catalog/launchapi`
  - `{{MCP_CONNECTOR}}://catalog/creativeapi`

## 开户管理（→ customerapi_call）

| 业务动作 | path | method | body 要点 |
|----------|------|--------|-----------|
| 可用广告平台列表 | `/media/getPlatformList` | POST | {clientId:0} → 返回 platformId/platformName（第1步确认平台的权威来源） |
| **已配置开户类型清单** | `/adAccountApply/busList` | POST | {} → 返回 AdsBusName/AdsBusType/AdsType；**开户类型必须从此实时拉取，按平台 AdsType 过滤，禁止写死枚举** |
| 公司主体列表 | `/company/list` | POST | {pageNum,pageSize} |
| 公司主体新增 | `/company/add` | POST | 主体字段 |
| 行业字典 | `/data/industrys` | POST | {} |
| 媒体代理渠道 | `/media/getMediaList` | POST | {platformId,clientId:0}（dev 环境部分平台可能为空，属正常） |
| 代理类型 | `/data/agentTypes` | POST | {adsType} |
| 绑定字典 | `/data/binds` | POST | {bindType:1} |
| TikTok 国家预设 | `/tiktok/country/preset/list` | POST | {page,pageSize}（TikTok 投放国家专属字典） |
| TikTok 时区 | `/tiktok/common/timezones` | POST | {}（TikTok 时区专属字典） |
| 提交 FB 开户 | `/adAccountApply/facebook` | POST | 收集字段 + adsBusType(来自 busList) |
| 提交 TikTok 开户 | `/adAccountApply/tiktok` | POST | 收集字段 |
| 提交 Google 开户 | `/adAccountApply/google` | POST | 收集字段 |
| 提交 Kwai 开户 | `/adAccountApply/kwai` | POST | 收集字段 |
| 提交其他媒体 | `/adAccountApply/add_otherMedia` | POST | 收集字段 |
| 开户申请列表 | `/adAccountApply/v2/list` | POST | {pageNum,pageSize} |
| 开户状态概览 | `/adAccountApply/v2/situation` | POST | {status:"pending"} |
| 开户申请详情 | `/adAccountApply/detail` | POST | {id,adsType} |

## 账户管理（→ customerapi_call）

| 业务动作 | path | method | body 要点 |
|----------|------|--------|-----------|
| 账户列表(多平台) | `/adAccount/v2/list` | POST | {pageNum,pageSize,adsTypeList} |
| 账户详情 | `/adAccount/detail` | POST | {id} |
| 操作前校验 | `/adAccount/check` | POST | {accountIdArr,isBalance,optType,adsType} |
| 充值工单 | `/adAccountOpt/balanceInc` | POST | {adsType,securityCode,idArr:[{id,amount}]} |
| 减款工单 | `/adAccountOpt/balanceSubstract` | POST | {id,amount,reason,securityCode} |
| 清零工单 | `/adAccountOpt/balanceClear` | POST | {adsType,securityCode,idArr:[id]} |
| 转款工单 | `/adAccountOpt/balanceMove` | POST | {id,targetId,amount,reason,securityCode} |
| 绑定/解绑 BM | `/adAccountOpt/bindPm` | POST | {adsType,yesOrNo,idArr,securityCode} |
| 绑定/解绑 Pixel | `/adAccountOpt/bindPixel` | POST | 同上 |
| 绑定/解绑 邮箱 | `/adAccountOpt/bindEmail` | POST | 同上 |
| 绑定/解绑 MCC | `/adAccountOpt/bindMCC` | POST | 同上 |
| 改名工单 | `/adAccountOpt/rename` | POST | {id,newAdAccountName,oldAccountName,reason} |
| 充值试算 | `/adAccountOpt/preSetPay` | POST | {adsType,idArr:[{id,amount}]} |
| 撤销工单 | `/work/saveTask` | POST | {workId,taskId,status:5,applySn,workDesc} |
| 操作记录 | `/adAccountOpt/v2/list` | POST | {pageNum,pageSize} |
| 工单状态 | `/adAccountOpt/v2/situation` | POST | {status:"pending"} |

## 财务管理（→ customerapi_call）

| 业务动作 | path | method | body 要点 |
|----------|------|--------|-----------|
| 我的财务账户 | `/finance/myAccount` | POST | {} |
| 付款订单 | `/finance/orders` | POST | {pageNum,pageSize} |
| 账单 | `/finance/bills` | POST | {pageNum,pageSize} |
| 发票 | `/finance/invoices` | POST | {pageNum,pageSize} |
| 返点 | `/finance/rebates` | POST | {pageNum,pageSize} |
| 钱包流水 | `/inner/post` | POST | {apiPath:"/client/account/walletLog",pageNum,pageSize} |
| 余额流水 | `/inner/post` | POST | {apiPath:"/client/account/balanceLog",...} |
| 币种列表 | `/client/getCurrencyList` | POST | {} |
| 汇率(按币种) | `/option/exchangeRate/findByCurrency` | POST | {currency:"USD"} |
| 支付渠道 | `/pay/getPayChannelList` | POST | {} |
| 交易记录 | `/adsPayOrder/getTransactionList` | POST | {pageNum,pageSize} |
| 支付状态 | `/adsPayOrder/getPayStatus` | POST | {orderId} |
| ⚠️钱包充值 | 在线支付下单被网关禁止 | — | 引导 UI 操作；MCP 仅可查到账 |

## 网关限制（调用前必读）
- customerapi 禁止：`/apiKey/**`、`/sys`、`/test`、`/user/login`、`/upload/**`、在线支付下单(`/adsPayOrder/pay` 等)。
- launchapi 禁止：`/test`、`/swagger-ui`、`/actuator`、OAuth 回调。
- creativeapi 仅允许 `/client/**`；上传用 `upload_material` 工具，不用 `/upload/**`。
- 资金工单 body 必带 `securityCode`；账号须已开启安全码。

## 原"占位 action"对照（已废弃）
早期猜测的 14 个细分动作（`开户/充值/清零/...`）**不准确**。真实形态是上面三张表的 path 网关。旧 Skill 里的 `action=<业务动作>` 占位已全部替换为 `customerapi_call(path=...)`。
