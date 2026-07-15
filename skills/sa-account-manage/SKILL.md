---
name: sa-account-manage
description: 广告账户管理 Skill。覆盖账户查询、充值、减款、清零、转款、绑定/解绑(BM/Pixel/邮箱/MCC/BC)、改名、撤销工单。所有写操作均生成工单(需 securityCode)，提交后运营审核。调用走 customerapi_call(path, method, body)（body camelCase）。
trigger: 查账户、账户操作记录、账户充值、充值、减款、转款、清零、绑定BM、绑定Pixel、绑定邮箱、绑定MCC、解绑、改名、撤销工单、账户详情
---

# 账户管理 Skill

## 通用前置
- **查询目标账户**：先 `customerapi_call(path="/adAccount/v2/list", method="POST", body={"pageNum":1,"pageSize":20,"adsTypeList":[<平台类型>]})` 让用户选账户（拿到账户 id）。
- **资金类工单需 securityCode**：用户必须已开启安全码；调用 body 带 `securityCode`。未开启则提示先去系统开启。
- **操作前校验**：`customerapi_call(path="/adAccount/check", method="POST", body={"accountIdArr":[<id>],"isBalance":1,"optType":<操作类型>,"adsType":<平台类型>})` 返回是否可操作。

## 闭环流程（按操作类型分支）

### A. 账户查询
- 列表（多平台）：`/adAccount/v2/list`（body: pageNum, pageSize, adsTypeList）
- 详情：`/adAccount/detail`（body: {"id":<账户id>}）
- 操作记录：`/adAccountOpt/v2/list`（body: pageNum, pageSize）
- 工单状态概览：`/adAccountOpt/v2/situation`（body: {"status":"pending"}）

### B. 充值（提交充值工单）
`customerapi_call(path="/adAccountOpt/balanceInc", method="POST", body={"adsType":<平台类型>,"securityCode":"******","idArr":[{"id":<账户id>,"amount":<金额>}]})`
- 单账户：idArr 一个元素；多账户：多个元素（统一/分账户金额在 body 中分别给）。
- 试算（可选）：`/adAccountOpt/preSetPay`（同结构，返回备用金计算公式）。
- 提交后生成工单 → 运营审核后到账。

### C. 减款（仅单账户）
`customerapi_call(path="/adAccountOpt/balanceSubstract", method="POST", body={"id":<账户id>,"amount":<金额>,"reason":"减款","securityCode":"******"})`
- 规则：减款金额建议 ≤ 账户余额的 2/3，避免工单受理失败；只能单账户，不能批量。

### D. 清零（提交清零工单）
`customerapi_call(path="/adAccountOpt/balanceClear", method="POST", body={"adsType":<平台类型>,"securityCode":"******","idArr":[<账户id>]})`
- **清零失败自检（提交前必查）**：
  1. 账户若无成功充值工单/成功转款工单 → 无法发起清零；
  2. 清零被驳回 → 不能再次发起，需新成功充值工单或联系业务；
  3. 清零未处理完成 → 无法再次发起；
  4. 清零成功后 → 需新成功充值工单才能再次发起。
- 时效：正常 1-2 工作日；无消耗最快 15 分钟内。

### E. 转款（同平台账户间）
`customerapi_call(path="/adAccountOpt/balanceMove", method="POST", body={"id":<源账户id>,"targetId":<目标账户id>,"amount":<金额>,"reason":"转款","securityCode":"******"})`
- 规则：只能相同平台账户间转移。

### F. 绑定 / 解绑（生成工单，需 securityCode）
- BM：`/adAccountOpt/bindPm`（body: {"adsType":1,"yesOrNo":1,"idArr":[<id>],"securityCode":"******"}）— yesOrNo=1 绑定 / 0 解绑；meta 用
- Pixel：`/adAccountOpt/bindPixel`（同结构）— meta 用
- 邮箱：`/adAccountOpt/bindEmail`（同结构）— Google 用
- MCC：`/adAccountOpt/bindMCC`（同结构）— Google 用
- TikTok 绑定 BC：通过开户流程 BC ID 字段；解绑 BC 走对应工单（如系统支持）

> 绑定前置：必须先绑定成功才能解绑；未绑定则系统无法解绑。

### G. 改名
`customerapi_call(path="/adAccountOpt/rename", method="POST", body={"id":<账户id>,"newAdAccountName":"新名称","oldAccountName":"旧名称","reason":"改名"})`

### H. 撤销审核中工单
`customerapi_call(path="/work/saveTask", method="POST", body={"workId":"...","taskId":"...","status":5,"applySn":"...","workDesc":"撤销该工单"})`
- 网关仅允许 status=5；workId/taskId/applySn 从 `/adAccountOpt/v2/list` 获取。

## 广告创建（launch 服务）纪律
> 广告创建/投放走 **launchapi_call**（与上方 customerapi_call 账户管理是两套服务）。本段为**强制纪律**，覆盖所有平台（FB/TikTok 等）。不单独成 Skill 文件，统一并入本账户管理 Skill 维护。

**铁律（最高优先级）**：
1. **只调 MCP，禁止脑补/想象**：所有字段名、端点、枚举值、下拉选项、预填值，**一律来自真实 MCP 返回**。agent 不得凭模型知识、截图 OCR 或文档自行推断、补默认值。截图只用于"对照发现遗漏"，不能当成字段来源。
2. **先问平台 + 广告类型(sceneType)**：未确认前绝不拉 schema、绝不建。sceneType 必须 `GET /batch/modules/schema?platform=&sceneType=` 实测验证后才列为可选项，不凭记忆列。
3. **字段全量渲染，不删减**：schema 返回的每一个模块/字段都必须展示，不得"精简"只留核心必填。
4. **下拉选项必须 MCP 同步/读取**：apps/identities/regions/options 先 sync 后取自真实返回；返回空则如实标"待提供/找运营"，不补。
5. **字段值由用户指定**：agent 不替用户默认填任何值（appId/预算/文案/地区等）。
6. **BC ID 仅 TikTok**：BC ID 字段只在 TikTok 渲染；其余渠道无 BC ID。
7. **提交闸门**：先预览完整 payload，等用户明确回"提交"才调写接口。

**流程**：
- Phase 0 鉴权 + 平台 OAuth 检查：`/user/getInfo`；launch 侧 `GET /{platform}/ad-accounts` 确认真实可投放账户，空则 `POST /{platform}/ad-accounts/sync-all`（同步非写）。dev 环境 Facebook OAuth 常为空 → 如实暴露阻塞，不伪造。
- Phase 1 问闸门：平台 + sceneType（MCP 验证有效类型）。
- Phase 2 取完整 schema：`GET /batch/modules/schema?platform=&sceneType=` → 渲染全量字段。
- Phase 3 同步真实下拉源（如 TikTok：`/batch/tiktok/apps`、`/batch/tiktok/identities`、`/batch/tiktok/regions`；sync 常 60s 限流但 GET 不受限）。
- Phase 4 渲染零编造表单（下拉=真 MCP 值，输入留空）。
- Phase 5 预览 payload + 等"提交"。

**端点**：批量创编 `POST /batch/tasks` → `PUT /batch/tasks/{id}` → `POST /batch/tasks/{id}/submit`；单对象 `POST /{platform}/ads/{adAccountId}/campaigns|adsets|ads|adcreatives`。

## 回执
- 提交工单后告知：工单号 + 操作类型 + 预计时效 + "运营审核后生效"。
- 用 `/adAccountOpt/v2/situation` 跟踪状态。

## 调用约定
- 所有 customerapi_call 目录端点 POST，body camelCase。
- 资金工单 body 必带 securityCode；账号须已开启安全码。
- 调用前可读 `{{MCP_CONNECTOR}}://catalog/customerapi` 核对路径。
- 禁止前缀：`/apiKey/**`、`/adAccountOptApi`、`/upload/**`。
