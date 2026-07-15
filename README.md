# 广告投放管理 Agent Skill 包

本包包含 4 个标准 Agent Skill，统一对接广告投放 MCP（{{MCP_CONNECTOR}}），覆盖**开户 / 账户管理 / 财务 / 常见问题排查**四大场景，内建「MCP 诚实性」纪律（只调真实返回、禁止脑补字段）。

## 目录结构

```
ad-skills/
├── skills/
│   ├── sa-account-opening/SKILL.md   # 开户管理（账户维度闭环）
│   ├── sa-account-manage/SKILL.md    # 账户管理 + 广告创建纪律
│   ├── sa-finance/SKILL.md           # 财务管理
│   └── sa-faq/SKILL.md               # 系统常见问题排查
├── README.md
└── references/
    ├── references-sa-mcp-tool-mapping.md # MCP 接口 ↔ Skill 工具映射
    └── references-mcp-honesty-rule.md     # 全局「禁编造」铁律（最高优先级）
```

## 安装方式

将本仓库的 `skills/` 目录复制到你的 Agent 项目的 skills 目录，并在 Agent 平台配置 MCP 连接器（见 INSTALL_PROMPT.md），重启平台后 4 个 Skill 即生效。

## 各 Skill 功能速览

| Skill | 触发词 | 核心能力 |
|-------|--------|----------|
| `sa-account-opening` | 开户 / 开账户 / 开 FB·TikTok·谷歌·Kwai 户 | Phase0 鉴权 → Phase1 列已配置开户类型（按媒体渠道分组、不写死）→ 动态表单 → 提交闸门（等「提交」才创建） |
| `sa-account-manage` | 查账户 / 充值 / 减款 / 清零 / 转款 / 绑定解绑（BM·Pixel·邮箱·MCC·BC）/ 改名 / 创建广告 | 账户查询、资金操作（工单 + securityCode）、绑定解绑；内建广告创建纪律（只调 MCP、字段全量、BC ID 仅 TikTok、禁脑补） |
| `sa-finance` | 钱包 / 可用余额 / 账单 / 发票 / 返点 / 退款 / 汇率 | 财务账户查询、付款订单 / 账单 / 发票、充值记录与审核、退款流程；实际充值走系统 UI（MCP 网关禁止在线支付下单） |
| `sa-faq` | 报错 / 失败 / 连不上 / MCP / 同步失败 | 业务类失败排查（清零 / 绑定前置 / 余额不足 / 到账差异 / 退款）+ 系统层排查（信任状态 → 重启 → 查连接器配置） |

## 全局纪律（已内建每个 Skill + 跨项目铁律）

所有数据必须来自真实 MCP 返回，**严禁编造**：

1. 选项 / 枚举必须 MCP 验证，不能凭模型知识列出；
2. 字段值由用户从 MCP 真实可选项明确指定，agent 不替填；
3. 某下拉 MCP 返回空时如实标注「待提供 / 找运营」，不得从文档补值；
4. 生成 payload 前逐字段说清来源（MCP 接口名 + 返回值）；
5. 所有写操作（提交 / 工单）须用户明确回复「提交」才执行。
