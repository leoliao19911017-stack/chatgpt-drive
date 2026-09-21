# chatgpt-drive 🚗

**让 AI Agent 可靠驱动网页版 ChatGPT：需求讨论、方案规划、生图，全程无需 API Key。**

> 💡 核心思路：**贵的模型出脑，便宜的模型出力。**
> 需求讨论、规划、生图这类「重脑力活」交给网页版 ChatGPT（高级模型 + 图像生成，订阅内随便用，不烧 API token）；
> 日常执行、文件操作、流程编排交给 GLM-Flash / Haiku 这类便宜模型。
> 一句 API 请求都不用发，token 成本直接砍掉一个数量级。

## 这解决什么问题

| 痛点 | 本技能的答案 |
|---|---|
| 高级模型 API 按 token 计费，长讨论一轮几块钱 | 驱动网页版 ChatGPT，订阅内无限用，0 API 成本 |
| Agent 调 ChatGPT 网页经常「看起来发了其实没发」 | 防乐观渲染三件套：30s 落地验证 / reload 真相 / 独特子串检查 |
| 生图不知道什么时候完成、怎么拿下来 | fragment 收割法判定完成 + 同步 canvas 下载（实测唯一可靠配方） |
| 长讨论一轮丢一次，成果蒸发 | 每轮 `--out` 自动落盘存档，线程丢了成果还在 |
| 贵模型干杂活，钱包哭泣 | 杂活全交给便宜模型执行本 skill，GPT 只出脑 |

全部命令与判据来自 2026-09 **真实生产会话**：四轮深度讨论（场景穷举 → 产品定义 → 成本定价 → 营销 GTM）产出完整方案文档，外加三张概念图全流程跑通。

## 工作原理

```
┌──────────────────┐         ┌─────────────────────┐
│ 便宜执行模型       │  浏览器  │   网页版 ChatGPT      │
│ (GLM/Haiku/Claude)│ ──────▶ │   高级模型 + 生图     │
│ 跑本 skill 编排    │ daemon  │   订阅内无限量        │
└──────────────────┘         └─────────────────────┘
        │
        └── 每轮产出自动落盘存档 → 下一轮压缩回填 → 讨论越滚越准
```

Agent 通过 gstack browse（playwright 有头浏览器）操作你**已登录的** ChatGPT 会话。所有可靠性问题——发送丢失、完成判定、生图收割——都有实测过的判据和恢复 SOP。

## 安装

前置：

- 本机已装 [gstack browse](https://github.com/gstack-io)（Claude Code 生态的浏览器技能，`~/.claude/skills/gstack/browse/`）
- 该浏览器已登录 ChatGPT

安装本技能：

```bash
# Claude Code / OpenClaw 用户：拷进技能目录
git clone https://github.com/leoliao19911017-stack/chatgpt-drive.git
cp chatgpt-drive/SKILL.md ~/.claude/skills/chatgpt-drive/SKILL.md
```

验证：

```bash
B="$HOME/.claude/skills/gstack/browse/dist/browse"
"$B" --headed status   # Status: healthy 即就绪
```

## 快速上手

把需求丢给你的 Agent，它就会按 SKILL.md 的协议执行：

> 「用 chatgpt-drive 帮我和 GPT 讨论一下这个产品该怎么做，先穷举使用场景」

> 「画一张智能家居概念图」

Agent 会自动：探活 → 发消息 → 30s 验证落地 → 轮询完成 → 抽取存档 → 生图收割下载 → 交付文件。

## 实战验证

- ✅ 四轮需求讨论（R1 场景穷举 / R2 产品定义 / R3 成本与定价 / R4 营销 GTM）→ 输出完整产品方案文档
- ✅ 3 张概念图生成 + 自动下载归档
- ✅ 静默丢信 2 次实弹恢复（§2 SOP 就是当时总结的）

## 踩坑精华（为什么这个 skill 值钱）

| 坑 | 后果 | 解法 |
|---|---|---|
| ChatGPT 乐观渲染 | 消息显示已发送，服务端根本没收到 | reload + 独特子串检查，别信 UI |
| 长线程消息计数 | 数 user 消息判断落地，永远对不上 | 虚拟化只挂载最后 ~2 条，改用子串 hit |
| 密集长文静默丢 | 连丢 2 次还不知道 | 单条 <3000 字 + 45s 后重发 |
| 异步下载假成功 | fetch/blob 写出 0 字节文件 | 同步 canvas drawImage（js 不 await promise） |
| 完成误判 | stop 按钮消失≠完成 | 三信号联合判定（n/len/stop） |

## 云端 Agent？

你的 Agent 跑在无桌面服务器上、ChatGPT 登录态在本地工作站？SKILL.md 附录给了反向 SSH 隧道桥接方案（含保活自愈脚本设计），本技能已在「云 Agent → 反向隧道 → 本地有头浏览器」架构下生产运行。

## 红线

- 绝不向 ChatGPT 发送 IP、token、密钥、内网域名、客户敏感信息
- 每轮讨论落盘存档
- 工具不可达时如实挂起，不假装执行

## 交流

- 📺 全流程演示视频（飞书对话 → Agent 调度 → GPT 生图 → 交付）正在路上。

- **进群交流 / 获取最新版**：见视频简介或评论区入口。

## License

MIT
