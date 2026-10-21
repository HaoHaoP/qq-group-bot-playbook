# 部署

从一台空服务器到群里能回话，约 40 分钟，其中扫码登录那步必须本人做。

## 前置

- 一台云服务器（买哪台见下），系统 Ubuntu 22.04 或更新
- Docker 与 compose 插件
- 一个专门的 QQ 小号（不要用主号）
- DeepSeek API Key

## 买哪台

按 2026-10 的公开价格， cheapest 不是看起来最便宜的那个：

| 方案 | 活动价 | 续费 | 目录价（API 实查） | 判断 |
|---|---|---|---|---|
| 轻量 2核2G / 200M峰值 / 40G | 38 元/年（新人秒杀）、68（新人常规）、79（老用户） | **默认按目录价 459 元/年**，活动实例才有折扣，且不支持跨规格同价续费 | 459 元/年 | 首年便宜，第二年翻十倍 |
| **ECS 经济型 e 2核2G / 3M固定 / 40G ESSD Entry** | **99 元/年，新老同享、续费同价，锁至 2029-03-31** | 99 元/年 | 千元级 | **推荐** |
| ECS 通用算力 u1 2核4G / 5M / 40G | 199 元/年，新老同享、续费同价 | 199 元/年 | 2366 元/年 | 内存紧时升级到这一档 |

要点：

1. **99 元档必须在活动入口下单**。普通购买页看到的是目录价（`ecs.e-c1m1.large` 配 5M 带宽查到 1543.74 元/年），差十倍以上，买错入口就白花。
2. 3M 固定带宽对群机器人绰绰有余——消息体极小，带宽不是瓶颈，别为 200M 峰值付费。
3. 2G 内存跑 NapCat + AstrBot 是紧的（协议端 500-900MB、平台 300-500MB），**装完第一件事加 2G swap**，见下。真的 OOM 了再升 4G 档，ECS 支持变配，轻量不支持。
4. 地域选华南（广州 `cn-guangzhou` 或深圳 `cn-shenzhen`），离得近、延迟低；机房 IP 对 QQ 风控是同等待遇，选近的即可。
5. 不用备案：不占 80/443、只用 IP 加非标准端口（6185 / 6199 / 6099）访问，不涉及域名备案。

```bash
# 加 2G swap，防 OOM
sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## 一、起容器

`deploy/docker-compose.example.yml` 里两个服务：

```yaml
services:
  napcat:
    image: mlikiowa/napcat-docker:latest
    container_name: napcat
    restart: unless-stopped
    environment:
      - NAPCAT_GID=0
      - NAPCAT_UID=0
    ports:
      - "6099:6099"   # WebUI，仅本机访问
    volumes:
      - ./data/napcat:/app/napcat/config
      - ./data/qq:/app/.config/QQ

  astrbot:
    image: soulter/astrbot:latest
    container_name: astrbot
    restart: unless-stopped
    ports:
      - "6185:6185"   # 管理面板
      - "6199:6199"   # OneBot 反向 WebSocket
    volumes:
      - ./data/astrbot:/AstrBot/data
```

两个镜像都在持续更新（NapCat 百万级拉取量），**不要用 `latest` 上生产**——第一次跑通后把 tag 固定成具体版本，避免某天 QQ 客户端更新把协议端带崩。

## 二、NapCat 登录

1. 浏览器打开 `http://<服务器IP>:6099/webui`，用日志里的 token 登录。
2. 扫码登录小号。
3. 进「网络配置」新增 **WebSocket 客户端（反向）**：
   - URL：`ws://astrbot:6199/ws`（同 compose 网络内用服务名）
   - Token：自己设一个，与下一步 AstrBot 里填的一致
4. 保存后确认日志出现连接成功。

## 三、AstrBot 接入

1. 打开 `http://<服务器IP>:6185`，首次进入设管理员密码。
2. 消息平台 → 新增 → **QQ（aiocqhttp）**：
   - 反向 WebSocket 主机：`0.0.0.0`
   - 反向 WebSocket 端口：`6199`
   - Token：与 NapCat 中一致
3. 模型供应商 → 新增 DeepSeek，填 API Key，模型选 `deepseek-flash`。
4. **关闭思考模式**：DeepSeek 的 `thinking` 默认为 enabled 且强度 high，不关会让输出 token 翻几倍。在供应商的请求参数里加 `{"thinking": {"type": "disabled"}}`。
5. 把机器人拉进群，@ 它说一句话，确认能回。

## 四、装限额插件

1. 把 `QingchenWait/astrbot_plugin_token_controller` 克隆到 `data/astrbot/plugins/`。
2. AstrBot 面板 → 插件 → 重载。
3. 在插件配置页填：

| 配置项 | 建议值 | 说明 |
|---|---|---|
| `limited_groups` | 你的群号 | 留空则不限流；群里发 `/sid` 可查群号 |
| `daily_token_limit` | 见「预算口径」换算 | 单位 token |
| `user_daily_token_limit` | 按人数摊 | 防单人刷 |
| `over_limit_policy.action` | `fallback_provider` 或 `stop_llm` | 到量换便宜模型，或直接停 |
| `refresh_time` | `00:00` | 与 DeepSeek 计价日对齐 |
| `block_message` | 自定义 | 超限时回的话，关掉 `send_block_message` 则静默 |

4. 群策略里建议同时打开「仅通过 @bot 触发」和「稳定前缀提升缓存命中（DeepSeek 专用）」。

## 五、验收清单

按顺序验，任一条不过就别往下走：

1. `/stats` 能返回当前会话的 token 用量，且「输入(缓存)」那一栏不是 0——说明缓存命中生效了。
2. 群里不 @ 它时，它不应答（除非你另配了主动型能力）。
3. 在插件里把 `daily_token_limit` 临时改成 1000：再发一句，按 `over_limit_policy` 应该换模型或静默，而不是照常回复。
4. 改回正常值，重启容器，用量统计不清零（插件按 ProviderStat 统计，落库在 AstrBot 数据目录）。
5. 平台账单与 `/stats` 的 token 数对得上（差异 >5% 时查下面「故障」第 4 条）。

## 六、故障对照

| 现象 | 先查 |
|---|---|
| NapCat 反复掉线 | 账号被风控：换小号重扫；机房 IP 是已知诱因 |
| AstrBot 显示平台在线但收不到消息 | 反向 WS 的 host/port/token 三处是否一致，容器网络是否互通 |
| 回复正常但 `/stats` 为 0 | 供应商没返回 usage；确认模型名是 `deepseek-flash` 而不是自定义中转名 |
| 账单明显高于统计 | 高峰时段调用（单价翻倍）、插件外部调用、或检索接口按次计费未计入 |
| 回复又长又贵 | 思考模式没关；或上下文没裁剪 |

## 未验证

- AstrBot 在容器内是否需要关闭多进程模式（部分教程提到多进程下插件不生效），按实际版本日志判断。
- 主动型能力（Proactive Agent）与「问句没人接时接一句」不是同一件事：前者偏定时任务式主动反馈，后者需要 `smart_wake` 这类监听未唤醒消息的插件。两者叠加的真实成本未实测。
