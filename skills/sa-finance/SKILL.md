---
name: sa-finance
description: 财务管理 Skill。覆盖钱包/可用余额查询、账单/发票/返点/付款订单查询、汇率与币种、充值记录与审核进度、退款流程，以及《客户端知识库Sop》5.2 财务常见问题自动应答。调用走 customerapi_call（body camelCase）。钱包实际充值走 UI（MCP 网关禁止在线支付下单）。
trigger: 钱包充值、充值、钱包余额、可用余额、余额不足、账单、发票、返点、对账、汇率、币种、退款、付款记录、财务账户
---

# 财务管理 Skill

## 一、查询类（MCP 可执行）

### 我的财务账户（余额/备用金）
`customerapi_call(path="/finance/myAccount", method="POST", body={})`
→ 返回钱包金额、可用余额、备用金等。

### 付款订单 / 账单 / 发票 / 返点
- 付款订单：`/finance/orders`（body: pageNum, pageSize）
- 账单：`/finance/bills`（body: pageNum, pageSize）
- 发票：`/finance/invoices`（body: pageNum, pageSize）
- 返点：`/finance/rebates`（body: pageNum, pageSize）
- 商务条款：`/finance/terms`
- 新增发票：`/finance/addInvoice`；新增凭证：`/finance/addVoucher`

### 财务流水（代理白名单）
- 钱包流水：`customerapi_call(path="/inner/post", method="POST", body={"apiPath":"/client/account/walletLog","pageNum":1,"pageSize":20})`
- 余额流水：`/inner/post` apiPath=`/client/account/balanceLog`
- 只读代理：`/inner/get`（apiPath 白名单：invoice/rebate/bill/prepayment）

### 汇率与币种
- 币种列表：`/client/getCurrencyList`
- 按币种查汇率：`/option/exchangeRate/findByCurrency`（body: {"currency":"USD"}）
- 汇率：`/data/rate`

### 充值记录与审核进度
- 支付方式列表：`/pay/getPayChannelList`、`/pay/getPayMethodList`、`/pay/getAuth`
- 交易记录：`/adsPayOrder/getTransactionList`（body: pageNum, pageSize）
- 支付状态：`/adsPayOrder/getPayStatus`（body: {"orderId":"xxx"}）

## 二、钱包充值（MCP 限制说明）
⚠️ **网关禁止**：在线支付下单（`/adsPayOrder/pay`、`orderPay`、汇付/合利宝/宝付等）与 `/upload/**` 均不可经 MCP 调用。
- 因此**钱包实际充值请引导用户在系统 UI 操作**：
  - 在线支付：美元(airwallex/photopay/pingpong/连连支付/万里汇)、人民币(汇付/易宝)
  - 线下打款：上传打款水单，运营认款后钱包+可用余额增加
- MCP 侧可**查询**充值是否到账：`/finance/myAccount`、`/adsPayOrder/getPayStatus`、`/finance/orders`。

## 三、退款流程（知识应答，见 5.2）
- 退款周期：媒体暂停广告 + 对每个账户发起清零工单 → 清零并暂停 7-10 天后可申请退款。
- 退款手续费：>1000 美金收 15 美金；≤1000 美金收 10 美金（合作服务费）。

## 四、可用余额不足——6 种解法（5.2 Q）
按顺序建议用户：
1. 打预付款 → 审核通过后钱包+可用余额增加；
2. 已打款但未到 → 查付款审核进度（或问运营）；
3. 其他账户有余额 → 提交**转款工单**（balanceMove，同平台）；
4. 有不用且余额的账户 → 提交**清零工单**增加可用余额；
5. 冻结金额过高 → 撤销审核中充值工单（见账户管理 Skill 的 /work/saveTask）或请运营驳回；
6. 未结算消耗过高 → 及时回款，回款后可用余额同步增加。

## 五、钱包 vs 可用余额（5.2 核心概念）
- **钱包**：存放预付款，用于月度账单结算；月度更新。
- **可用余额**：用于广告账户资金往来，实时更新，仅做系统资金风控（实际扣款以月度账单为准）。
  - 增加：预付款/账单回款、广告账户减款、清零成功
  - 减少：广告账户充值、钱包退款成功
- 可用余额公式（按客户类型）：预付 = 可用余额 − 冻结金额；预付实销 = +超额/临时超额/临时授信 − 冻结；后付 = +授信 + 超额/临时超额 − 冻结。

## 六、回执
- 查询类直接呈现返回数据（分页默认 pageNum=1, pageSize=20）。
- 涉及"需 UI 操作"的事项明确告知用户在系统页面完成，并给查询入口验证结果。

## 调用约定
- customerapi_call 均 POST，body camelCase。
- `/inner/get` 与 `/inner/post` 的 apiPath 必须在财务白名单内。
- 调用前可读 `{{MCP_CONNECTOR}}://catalog/customerapi` 核对路径。
