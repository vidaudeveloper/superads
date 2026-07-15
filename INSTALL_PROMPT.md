# 广告投放 Agent Skill 包 · 安装提示词

> 本包包含 4 个标准 Agent Skill，统一对接广告投放 MCP（{{MCP_CONNECTOR}}），覆盖**开户 / 账户管理 / 财务 / 常见问题排查**四大场景。内建「MCP 诚实性」纪律。
> 复制下面内容到 AI Agent 对话即可按步骤安装。

---

## 一、把 Skill 文件落位

将本仓库的 `skills/` 目录复制到目标 Agent 项目的 skills 目录：

```
<project root>/
├── skills/
│   ├── sa-account-opening/SKILL.md
│   ├── sa-account-manage/SKILL.md
│   ├── sa-finance/SKILL.md
│   └── sa-faq/SKILL.md
├── README.md
└── references/
    ├── references-sa-mcp-tool-mapping.md
    └── references-mcp-honesty-rule.md
```

## 二、配置广告投放 MCP 连接器

本包依赖 MCP 连接器 **{{MCP_CONNECTOR}}**（工具前缀 `mcp__{{MCP_CONNECTOR}}__*`）。

在目标 Agent 平台的连接器 / MCP 配置中添加：

```json
{
  "{{MCP_CONNECTOR}}": {
    "type": "sse",
    "url": "{{MCP_SSE_URL}}",
    "headers": {
      "api-key": "<你的 API Key>"
    }
  }
}
```

信任该连接器后重启平台，使 MCP 工具暴露。

## 三、验证安装

重启后，在对话里尝试：

- 说「**查账户**」「**开户**」「**钱包余额**」「**MCP 连不上**」等触发词，确认对应 Skill 被激活；
- 或确认 {{MCP_CONNECTOR}} 连接器状态为「已连接」，调用 `mcp__{{MCP_CONNECTOR}}__customerapi_call`（path=`/company/list`）能返回真实数据。

## 四、四个 Skill 能力速览

| Skill | 触发词 | 核心能力 |
|-------|--------|----------|
| `sa-account-opening` | 开户 / 开账户 / 开 FB·TikTok·谷歌·Kwai 户 | Phase0 鉴权 → Phase1 列已配置开户类型（按媒体渠道分组、不写死）→ 动态表单 → 提交闸门 |
| `sa-account-manage` | 查账户 / 充值 / 减款 / 清零 / 转款 / 绑定解绑 / 改名 / 创建广告 | 账户查询、资金操作（工单 + securityCode）、绑定解绑；广告创建纪律（只调 MCP、禁脑补） |
| `sa-finance` | 钱包 / 可用余额 / 账单 / 发票 / 返点 / 退款 / 汇率 | 财务查询、付款订单 / 账单 / 发票、充值记录与审核、退款；实际充值走系统 UI（网关禁在线支付） |
| `sa-faq` | 报错 / 失败 / 连不上 / MCP / 同步失败 | 业务类失败排查 + 系统层排查（信任状态 → 重启 → 查连接器配置） |

## 五、全局纪律（已内建每个 Skill，请务必遵守）

所有数据必须来自真实 MCP 返回，**严禁编造**：

1. 选项 / 枚举必须 MCP 验证，不能凭模型知识列出；
2. 字段值由用户从 MCP 真实可选项明确指定，agent 不替填；
3. 某下拉 MCP 返回空时如实标注「待提供 / 找运营」，不得从文档补值；
4. 生成 payload 前逐字段说清来源（MCP 接口名 + 返回值）；
5. 所有写操作（提交 / 工单）须用户明确回复「提交」才执行。

## 六、MCP 工具映射（速查）

| 工具 | 用途 |
|------|------|
| `mcp__{{MCP_CONNECTOR}}__customerapi_call` | 广告账户、报表、财务、开户、工单（body camelCase） |
| `mcp__{{MCP_CONNECTOR}}__launchapi_call` | 批量创编、投放查询、同步、状态变更 |
| `mcp__{{MCP_CONNECTOR}}__creativeapi_call` | 素材库（`/client/**`，body snake_case） |
| `mcp__{{MCP_CONNECTOR}}__upload_material` | 上传素材（预签名直传或 by-url） |

调用前可读 MCP Resource 获取完整路径与示例：`{{MCP_CONNECTOR}}://catalog/customerapi`、`{{MCP_CONNECTOR}}://catalog/launchapi`、`{{MCP_CONNECTOR}}://catalog/creativeapi`。

---

> 部署前请替换占位符：
> - `{{MCP_CONNECTOR}}` → 你的 MCP 连接器名称
> - `{{MCP_SSE_URL}}` → 你的 MCP SSE 端点
