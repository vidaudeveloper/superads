---
name: sa-faq
description: 常见问题排查 Skill。覆盖业务类失败排查（清零失败、绑定/解绑前置、可用余额不足、到账差异、退款）与系统/接口层排查（MCP 连不上、工具找不到、同步失败、权限）。优先用 customerapi_call 查询真实状态再给结论。
trigger: 报错、异常、失败、清零失败、绑定不了、解绑不了、余额不足、到账差异、退款、连不上、MCP、同步失败、查不到工具
---

# 系统常见问题 Skill

## 一、业务类失败排查（先查真实状态，再下结论）

### 清零失败
用 `customerapi_call(path="/adAccountOpt/v2/situation", method="POST", body={"status":"<状态>"})` 或 `/adAccountOpt/v2/list` 看工单状态，对照 4 条规则：
1. 无成功充值/转款工单 → 无法发起清零；
2. 被驳回 → 不能再次发起，需新充值工单或联系业务；
3. 未处理完成 → 无法再次发起；
4. 成功后 → 需新成功充值工单才能再发起。

### 绑定/解绑前置
- 必须先绑定成功才能解绑；未绑定则系统无法解绑。
- 平台对象：meta→BM/Pixel；Google→MCC/邮箱；TikTok→BC。
- 提交绑定/解绑工单走账户管理 Skill 的 `/adAccountOpt/bind*`。

### 可用余额不足
见财务管理 Skill「可用余额不足 6 种解法」（打预付款 / 查审核 / 转款 / 清零 / 撤冻结 / 回款）。

### 到账差异 / 钱包比实际付款少
- 钱包月度更新、可用余额实时更新（两者更新频率不同，属正常）。
- 少额多因手续费（按付款方式，在线支付页有说明）。
- 线下打款需上传水单/同步运营认款才到账。

### 退款相关
- 周期：清零+暂停 7-10 天后可申请；手续费 >1000 收 15、≤1000 收 10 美金。

## 二、系统 / 接口层排查

### MCP 连不上 / 工具找不到（mcp__{{MCP_CONNECTOR}}__* 不可用）
自检顺序：
1. 连接器管理 → 自定义连接器 → 确认 `{{MCP_CONNECTOR}}` 已**信任/启用**且状态绿点；
2. 彻底退出重启 Agent 平台（托盘也退，让本地代理进程 127.0.0.1:61087 重载）；
3. 确认 `mcp.json（位置因平台而异）` 含 `{{MCP_CONNECTOR}}` 条目（type:sse + url + headers.api-key）；
4. 确认 `连接器状态配置（位置因平台而异）` 的 `enabled` 数组含 `"{{MCP_CONNECTOR}}"`；
5. 网络：SSE 端点 `{{MCP_SSE_URL}}` 需返回 401（缺 api-key 头属正常），若超时/拒绝连接则检查网络或 VPN。

### 同步失败（投放数据看不到）
- TikTok：`customerapi_call` 不适用；用 `launchapi_call(path="/tiktok/ad-accounts/sync-all", method="POST", body={})` 一键同步，或单户 `/tiktok/ad-accounts/{adAccountId}/sync`。
- Facebook/Google：对应 `launchapi_call` 的 `.../sync-all` 接口。
- 同步后查：`/ad/queryTikTokAccountPage`(GET) 等投放查询接口。

### 权限 / 401 / 无数据
- 确认 api-key 有效且未过期；
- 确认操作账户属于当前登录客户（UserId 一致）；
- 资金类需账号已开启安全码（securityCode）。

## 三、回执
- 业务问题：给出「状态 + 原因 + 下一步动作」。
- 系统问题：给出「自检到第几步 + 具体修复动作」。

## 调用约定
- 业务状态查询走 customerapi_call（POST, camelCase）；
- 投放同步走 launchapi_call（sync-all 为 POST，查询为 GET）；
- 调用前可读 `{{MCP_CONNECTOR}}://catalog/customerapi` / `{{MCP_CONNECTOR}}://catalog/launchapi` 核对路径。
