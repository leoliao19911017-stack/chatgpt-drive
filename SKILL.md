---
name: chatgpt-drive
description: 通过有头浏览器可靠驱动网页版 ChatGPT 生图/讨论：多轮需求讨论、防静默丢信发送、完成轮询、回答存档、生图收割与下载。当收到"生图/出图/画一张/做概念图/海报素材"类需求时使用本技能。
---

# 网页版 ChatGPT 驱动手册

以下命令与判据全部经真实会话实测（四轮长讨论 + 多张生图全流程跑通）。

**适用场景**：让便宜的执行型模型（GLM / Haiku / 小参数模型）驱动网页版 ChatGPT 干重活——需求讨论、方案规划、生图。贵模型出脑，便宜模型出力，token 花在刀刃上。

## ⚠️ 运行环境（先读）

- 前置：本机已安装 gstack browse（`~/.claude/skills/gstack/browse/dist/browse`），且该浏览器**已登录 ChatGPT**。
- 本技能的命令全部在**本机**执行，daemon 沿用已登录的浏览器会话。
- **先探活**：任何生图/讨论任务开工前先跑 `"$B" status`，`Status: healthy` 才继续。不可达时：**如实告知用户并挂起请求**，**严禁假装执行过或编造产物路径**。
- 产物统一存一个固定目录（如 `~/photos`），交付时上传或发文件，不直接报本地路径完事。

## 0. 前置

```bash
B="$HOME/.claude/skills/gstack/browse/dist/browse"
```

- **所有命令必须带 `--headed`**（无头模式过不了 ChatGPT 反自动化）。前提：浏览器已登录 ChatGPT，daemon 沿用既有会话。
- 若报 `existing daemon has different config (proxy/headed mismatch)`：先 `"$B" disconnect` 再带 `--headed` 重跑。
- `"$B" --headed status` 查看 Status/URL/Tabs/PID，确认当前页面。

## 1. 发消息三步

```bash
"$B" --headed fill "#prompt-textarea" "消息内容"
"$B" --headed press Enter
```

- fill 自带聚焦；若 Enter 未送出，先 `click "#prompt-textarea"` 再 press。
- bash 传中文长文没问题；但**单条尽量 <3000 字**——密集长文更容易被服务端静默丢弃（实测连续丢 2 次，裁短后第 3 次成功）。
- js 表达式里内层字符串用单引号，bash 外层用双引号，实测无冲突。

## 2. 真相只有 reload（ChatGPT 乐观渲染）

消息显示"已发送"≠服务端收到。**发出后 30s 内必须验证落地**。

**持久化检查（独特子串法）**——⚠️ 不要用 `[data-message-author-role="user"]` 的**数量**判断：长线程虚拟化只挂载最后 ~2 条，计数会骗你。用子串命中：

```bash
"$B" --headed js "[...document.querySelectorAll('[data-message-author-role=\"user\"]')].map(e=>e.innerText).join('').includes('本轮消息里独一无二的短语')"
```

返回 `false` 或查不到 = 没落地。需要看真相时 `"$B" --headed reload` 再查。

**静默丢信识别与恢复**（实测 SOP）：
1. 症状：出现「正在思考」占位（assistant len=4）或 len=0，stop=false，卡住 >2–3 分钟无新内容。
2. `reload` → 子串检查确认消息是否真的没了（user 计数回退）。
3. **等 45 秒**再重发（服务端可能在处理，立刻重发会叠加）；重发时裁短/精简。
4. 仍失败 → 换新表述再发一次。

## 3. 完成轮询

**状态表达式**（一次拿全三个信号）：

```bash
"$B" --headed js "JSON.stringify({n:document.querySelectorAll('[data-message-author-role=\"assistant\"]').length,len:(()=>{const a=[...document.querySelectorAll('[data-message-author-role=\"assistant\"]')];return a.length?a[a.length-1].innerText.length:0})(),stop:[...document.querySelectorAll('button')].some(b=>/stop|停止/i.test(b.getAttribute('aria-label')||''))})"
```

| 判据 | 含义 |
|---|---|
| `stop=true`，n/len 不变 | **思考中**（旧消息仍是最后一条，别误判为完成） |
| `stop=false` + `len>300` + 连续两次轮询 len 相同 + n 已增长 | **完成** |
| `stop=false` + len≤4 或 0 持续数分钟 | **静默丢信**，走 §2 恢复 |

**轮询方式**：用后台任务跑 until 循环（完成即退出，收到一次完成通知），设最大轮数防挂死：

```bash
for i in $(seq 1 60); do
  R=$("$B" --headed js "<状态表达式>" 2>/dev/null)
  echo "$R" | grep -q '"stop":false' && echo "$R" | grep -qv '"len":4' && echo "$R" | grep -qv '"len":0' && { echo "DONE $R"; break; }
  sleep 10
done
```

（更稳的做法：先记下发出前的 `len` 基线，完成条件=stop=false 且 len>基线 且连续两轮相同。）

## 4. 回答抽取存档

`js` + `--out` 直落文件（这就是每轮讨论的存档）：

```bash
"$B" --headed js "[...document.querySelectorAll('[data-message-author-role=\"assistant\"]')].slice(-1).map(e=>e.innerText).join('\n\n---\n\n')" --out "C:/tmp/gpt_r1.txt"
```

- 取最后 N 条把 `slice(-1)` 改 `slice(-N)`。
- 落盘后**必须 `ls -la` 看大小 + 抽查内容**，空文件=抽取失败（常见原因见 §6）。
- 存档命名约定：`gpt_r1_主题.txt`、`gpt_r2_主题.txt`…放任务专属目录。

## 5. 生图：收割与下载

**完成判定（fragment 收割法）**：

```bash
"$B" --headed js "JSON.stringify([...new Set([...document.querySelectorAll('img')].map(i=>{const m=(i.src||'').match(/id=(file_[a-f0-9]+)/);return m?m[1].slice(13,19):null}).filter(Boolean))])"
```

- 开工前先跑一次记下**已知 fragment**（旧图）；之后出现 **≥3 张 img 共享同一个未知新 fragment** = 本轮生成完成。
- 生图排队可能要几分钟，轮询同 §3 节奏。

**下载（同步 canvas，唯一可靠配方）**：

```bash
"$B" --headed js "const frag='新fragment';const img=[...document.querySelectorAll('img')].find(x=>(x.src||'').includes(frag));const c=document.createElement('canvas');c.width=img.naturalWidth;c.height=img.naturalHeight;c.getContext('2d').drawImage(img,0,0);c.toDataURL('image/png')" --out "C:/tmp/shot1.png"
```

- ⚠️ **browse js 不 await promise**：fetch/blob/async IIFE 全部静默失败产出 0 字节文件，**必须用上面的同步 canvas**。
- 下载完 `ls -la` 验字节数，再目检图片内容（是否熔脸/文不对题）。
- 交付前把图发给用户，**不能只报一个本地路径就当交付**。

## 6. 已踩坑速查

| 坑 | 解法 |
|---|---|
| bash 双引号内联 PowerShell 吞 `$var`（`.Length` 处报解析错） | heredoc 加引号定界符写 .ps1：`cat > x.ps1 <<'EOF'`，再执行 |
| bash 处理中文文件名乱码 | 目录名用 ASCII；中文文件名只经 Write 工具或 PowerShell 操作 |
| js 里写复杂 `case`/嵌套引号导致 bash 报 `unexpected EOF` | 拆简单表达式；避免 case 语句 |
| user 消息计数对不上 | 虚拟化只挂载最后 ~2 条，用子串 hit 判断 |
| 以为发成功了其实没发 | 一切以 reload 后的子串检查为准，别信乐观渲染 |
| daemon 配置不匹配报错 | `"$B" disconnect` 后重跑 |

## 7. 红线

- **绝不向 ChatGPT 发送服务器 IP、token、密钥、内网域名、客户敏感信息**——提示词只谈产品能力、公开行情与结构化需求。
- 每轮回答必须落盘存档，防止线程丢失后讨论成果蒸发。
- 工具不可达时如实挂起请求，不假装执行。

## 8. 多轮讨论协议（实测高效打法）

1. **R1 场景穷举** → R2 定方向/产品定义 → R3 数字（成本/定价/账）→ R4 执行（营销/GTM/排期）。每轮只解决一层。
2. 每条提示词结构：**上轮结论编号回顾（压缩到几行）→ 本轮编号问题 → 明确输出格式要求（"用表格、给具体数字、不要泛泛而谈"）**。GPT 在被要求表格化+数字化时输出质量显著更高。
3. 节奏：发 → 30s 验证落地 → 轮询完成 → `--out` 存档 → 消化 → 把结论压进下一轮提示词。
4. 需要"汇报素材"时用生图（§5），概念图足够则不浪费时间生成视频。
5. 收尾：综合全部轮次产出完整方案文档；GPT 原话中引用的外部数据（法规/竞品价格）标注"需二次核实"，不直接当事实。

## 附录：云端 Agent 桥接本地浏览器（进阶部署）

如果你的 Agent 跑在云服务器上（无桌面、无浏览器），而 ChatGPT 登录态在本地工作站，可以搭一条反向 SSH 隧道：

1. 工作站开 OpenSSH Server（防火墙只放行回环）；
2. 工作站常驻 `ssh -N -R 127.0.0.1:22022:127.0.0.1:22 root@<服务器IP>`（SYSTEM/服务方式保活，断线自动重连）；
3. 服务器侧写一个包装脚本 `ws-browse`：把参数 base64 封装（防多层转义），经隧道 SSH 回工作站用 Git bash 解码执行本地 `browse --headed`，并固定 `BROWSE_STATE_FILE`/`BROWSE_PORT` 环境变量（browse 按 cwd 发现 daemon，不钉死会各起各的）；
4. 工作站放一个带超时探活 + 失败全栈清杀重启的保活脚本（ headed Chrome 必须在交互桌面会话，SSH 会话 Session 0 起不来）；
5. 产物用 `scp -P 22022` 从工作站拉回服务器。

细节坑：多层 shell 转义必丢层，一律本地写好脚本再 scp；workstation 侧严禁裸跑 browse（会在当前目录起野 daemon 占住 profile 锁）。
