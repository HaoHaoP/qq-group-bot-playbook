# QQ 群 AI 机器人 低成本落地手册

把 DeepSeek 接进 QQ 群，读全群消息、能回答问题、能择机插话，并且把花费压在每月个位数——用现成组件拼出来，不写框架代码。

这个仓不是另一个机器人项目，而是一份**可复现的部署与调参记录**：每个组件为什么选它、每个数字从哪来、哪些结论还没验证。

## 结论先行

| 层 | 选型 | 代价 |
|---|---|---|
| 协议端 | NapCat（OneBot v11，扫小号登录） | 机房 IP 有风控，掉号要换 |
| 平台层 | AstrBot（aiocqhttp 适配器） | 无 |
| 模型 | DeepSeek `deepseek-flash`，非思考模式 | 需显式关闭思考模式，否则输出翻几倍 |
| 限额 | `astrbot_plugin_token_controller` | 按 token 而非金额，见「预算口径」的换算 |
| 统计 | AstrBot 自带 `/stats` | 四路拆分：输入(缓存) / 输入(其他) / 输出 / 总量 |
| 服务器 | 2核2G 轻量（¥38–68/年档） | 不用函数计算，理由见 DECISIONS |

一个日均 300 条消息、每天约 20 次回话的小群，模型月支出约 2 元，加服务器约 8 元/月。

## 快速开始

1. 买一台 2核2G 轻量服务器，装 Docker。
2. `deploy/docker-compose.example.yml` 起两个容器：`napcat` + `astrbot`。
3. 打开 NapCat WebUI 扫码登录小号，配置 OneBot 反向 WebSocket 指向 AstrBot 的 `6199` 端口。
4. AstrBot 里加平台「QQ（aiocqhttp）」，主机填 `0.0.0.0`、端口 `6199`、token 与 NapCat 一致。
5. 把机器人拉进群，@ 一次确认通。
6. 装 `token_controller` 插件，配 `limited_groups` 与 `daily_token_limit`。

详细步骤与验收清单见 [docs/SETUP.md](docs/SETUP.md)。

## 仓内文档

- [docs/SETUP.md](docs/SETUP.md) — 从空服务器到群里能回话，含验收清单与故障对照
- [docs/BUDGET.md](docs/BUDGET.md) — 预算口径：单价、时段价差、token 与金额怎么换算、怎么对账
- [docs/DECISIONS.md](docs/DECISIONS.md) — 选型记录：为什么不用函数计算、为什么不自研框架也不自研插件

## 一句话经验

群消息是无限的，模型是按 token 计费的，两者之间必须有闸门。这个闸门最好由「该不该调用」控制，而不是由「说了多少字」控制——所以批量判定、@ 触发、上下文裁剪三件事，比换便宜模型更能省钱。

## 许可

文档与示例代码：AGPL-3.0。
NapCat、AstrBot、token_controller 各自遵循其自身许可。
