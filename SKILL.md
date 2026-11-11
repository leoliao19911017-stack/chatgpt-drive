---
name: chatgpt-drive
description: Use when 生图 or multi-round GPT web discussion is needed.
---

# 网页版 ChatGPT 驱动手册

以下命令与判据全部经真实会话实测（四轮长讨论 + 多张生图 + 云端桥接全流程跑通）。来源：https://github.com/leoliao19911017-stack/chatgpt-drive

**适用场景**：让便宜的执行型模型（GLM / Haiku / 小参数模型）驱动网页版 ChatGPT 干重活——需求讨论、方案规划、生图。贵模型出脑，便宜模型出力，token 花在刀刃上。收到「生图/出图/画一张/做概念图/海报素材」类需求时使用。

## ⚠️ 开工前必读（三条铁律）

1. **先探活，再动手。** 任何任务开工前先跑 `start-browse.sh` + `status`。不可达时**如实告知用户并挂起请求**，**严禁假装执行过或编造产物路径**。
2. **不擅自重启、不擅自杀进程。** 同机可能跑着别的 agent（OpenClaw 等）共用同一套 browse。**杀进程会误伤它们**。要重启先只读探活确认归属，有疑问问用户。
3. **绝不启用保活循环。** `ssh-tunnel/browse-daemon.ps1` 那个 `while($true)` 循环已被禁用（曾因杀名单过宽导致有头浏览器全天反复弹窗、严重干扰用户）。**保持禁用**，只用按需单次启动。

---

## 0. 前置

```bash
bash ~/.claude/skills/gstack/browse/start-browse.sh
export BROWSE_STATE_FILE="<USER_HOME>/.gstack/browse-state.json"
B="$HOME/.claude/skills/gstack/browse/dist/browse"
```

`start-browse.sh` 是**幂等**的：已在跑就复用，没跑才启（内部用 node 起 `server-node.mjs`，并补 `configHash` 让 CLI 肯复用）。

- **所有命令必须带 `--headed`**（无头模式过不了 ChatGPT 反自动化）。
- 若报 `existing daemon has different config`：state file 的 configHash 与当前 flag 不符。**别急着 `disconnect`**（会杀掉 daemon 拖累同机 agent）；先确认 `BROWSE_STATE_FILE` 指向的是不是你想复用的那个。
- 探活判据：`"$B" --headed status` 返回 `Status: healthy`。

## 0.1 ⭐ 共用同一个浏览器（2026-10-10 定稿）

**Hermes Agent 和本机 Claude Code 共用同一个 browse daemon，不要各开一个。**

| 项 | 值 |
|---|---|
| **state file** | `<USER_HOME>/.gstack/browse-state.json` |
| **daemon** | 由首条命令 spawn，孤儿进程，跨 Bash 存活 |
| **profile** | `.gstack/chromium-profile`（已登录 ChatGPT） |
| **代理** | 启动参数带 `--proxy-server=127.0.0.1:7897` |

**铁律：**

1. **永远 export `BROWSE_STATE_FILE="<USER_HOME>/.gstack/browse-state.json"`** —— 这是共用的那个。
   ⚠️ 历史遗留的 **`browse-web.json` 已废弃**，别再用了（用它 = 另起 daemon = 多弹一个浏览器窗口，用户明确要求避免）。
2. **窗口同一时间只有一个。** 一方用完不关，另一方直接接上去用。
3. **绝不 `disconnect` / `stop` / 杀 daemon** —— 会打断另一方正在进行的操作（见 §0.5）。
4. **共用意味着谦让**：操作前先 `status` 看当前 URL；发现线程页被切走了，**goto 深链回去**即可（`goto https://chatgpt.com/c/<id>`）。

**并发约定：同一时刻只允许一个 agent 驱动浏览器。** 需要接手时，先确认对方 `status` 里 URL ≥30s 无变化再动。

如果确实报 `existing daemon has different config`：**不要 `disconnect`**，先检查是不是漏了 `--headed`。

## 0.5 daemon 生命周期与静默铁律（2026-09-24 定稿，最高优先级）

> 背景：本机曾有一个 Startup 自启保活脚本 `ssh-tunnel/browse-daemon.ps1`（小小苏生图桥遗留），`while($true)` 每 60s 检查 daemon，不健康就**清杀整个 browse 栈再重拉 headed 窗口**——其杀名单含 `server-node.mjs` 和 `chromium-profile` 的 chrome，会误伤会话里正在用的 daemon。结果就是**浏览器每隔一段时间反复弹出、干扰用户使用电脑**（用户暴怒）。09-24 已由 Codex 停掉循环+禁用启动项（`browse-daemon.cmd.disabled`）。**任何会话都不得以任何形式恢复周期性保活。**

三条铁律：

1. **单窗口**：同一时间 browse 窗口只留一个。绝不反复开关、绝不乱杀 chrome/node 进程、绝不为了"干净"而重启 daemon。
2. **禁保活**：不得创建 Startup 项 / 计划任务 / while 循环来自动拉起 browse daemon。daemon 是**任务内按需**的存在，不是常驻服务。
3. **daemon 死了先请示**：daemon/窗口没了时**不得静默重开**（CLI 会自动 spawn 新窗口弹到用户脸上）——先告知用户，征得同意再重启。若用户手头不需要，改约时间或放弃浏览器路径。

正确姿势（保活禁用后的实测稳态）：

- **daemon 由 CLI 首条命令 spawn 后成为孤儿进程可跨 Bash 调用存活**（父进程消失不影响它）。首条命令前 export 两件套：
  ```bash
  export BROWSE_STATE_FILE="<USER_HOME>/.gstack/browse-state.json" \
         GSTACK_CHROMIUM_PATH="C:/Program Files/Google/Chrome/Application/chrome.exe"
  B="$HOME/.claude/skills/gstack/browse/dist/browse"; "$B" --headed ...
  ```
- **单条前台 Bash 调用 ≤120s 内完成一个完整动作**（goto→操作→验证）。`run_in_background`/被超时转后台的 shell 会失联 daemon；跨多条调用分步时靠孤儿 daemon 续命。
- **bun 跑不了 server-node.mjs**（`bun-polyfill.cjs` readonly property 崩溃），node 才是正确运行时——CLI 自己 spawn 的就是 node，别用 bun 手动常驻。
- **用户手关窗口 = daemon 死**（`handleChromiumDisconnect` 设计行为，窗口与 server 同生共死）。别猜，status 一查便知。
- **daemon 重启/页面丢失后，深链 goto 线程 URL 直达**（`goto https://chatgpt.com/c/<id>` → sleep 15 等水化 → `#prompt-textarea` 就绪）。侧栏 /c/ 链接渲染飘忽不可依赖；viewport 用 `--headed viewport 1680x950`（944px 窄屏侧栏会收起）。
- **静默原则**：本地能验证的（HTML/图片/文件）绝不碰浏览器；验证本地页面时复用现有窗口 goto，事毕导航回原线程页；操作频次能省则省。

## 0.6 本机环境速查

| 项 | 值 |
|---|---|
| browse CLI | `$HOME/.claude/skills/gstack/browse/dist/browse` |
| 启动脚本 | `bash ~/.claude/skills/gstack/browse/start-browse.sh`（daemon 真的没了才用，先请示） |
| daemon 现状 | `cat <USER_HOME>/.gstack/browse-state.json` |
| 代理 | Clash `127.0.0.1:7897`；**系统代理须开**（`ProxyEnable=1`，否则 Chrome 上不了 Google） |
| 截图 | `"$B" --headed screenshot /path/out.png` |

## 1. 发消息三步

```bash
"$B" --headed fill "#prompt-textarea" "消息内容"
"$B" --headed press Enter
```

- fill 自带聚焦；若 Enter 未送出，先 `click "#prompt-textarea"` 再 press。
- **单条尽量 <3000 字**——密集长文更容易被服务端静默丢弃（实测连续丢 2 次，裁短后第 3 次成功）。
- 中文传参用 base64 中转，避免多层转义丢层：

```bash
B64=$(printf '%s' "$PROMPT" | base64 -w0)
ssh ... "P=\$(echo $B64 | base64 -d); \"\$B\" --headed fill '#prompt-textarea' \"\$P\""
```

## 2. 真相只有 reload（ChatGPT 乐观渲染）

消息显示"已发送" ≠ 服务端收到。**发出后 30s 内必须验证落地**。

**持久化检查（独特子串法）**——⚠️ **不要**用 `[data-message-author-role="user"]` 的**数量**判断：长线程虚拟化只挂载最后 ~2 条，计数会骗你。用子串命中：

```bash
"$B" --headed js "[...document.querySelectorAll('[data-message-author-role=\"user\"]')].map(e=>e.innerText).join('').includes('本轮消息里独一无二的短语')"
```

返回 `false` = 没落地。需要看真相时 `"$B" --headed reload` 再查。

**静默丢信识别与恢复**：
1. 症状：「正在思考」占位（assistant len=4）或 len=0，stop=false，卡 >2–3 分钟无新内容。
2. `reload` → 子串检查确认消息是否真没了。
3. **等 45 秒**再重发（服务端可能在处理，立刻重发会叠加）；重发时裁短精简。
4. 仍失败 → 换新表述再发一次。

## 3. 完成轮询

**状态表达式**（一次拿全三个信号）：

```bash
"$B" --headed js "JSON.stringify({n:document.querySelectorAll('[data-message-author-role=\"assistant\"]').length,len:(()=>{const a=[...document.querySelectorAll('[data-message-author-role=\"assistant\"]')];return a.length?a[a.length-1].innerText.length:0})(),stop:[...document.querySelectorAll('button')].some(b=>/stop|停止/i.test(b.getAttribute('aria-label')||''))})"
```

| 判据 | 含义 |
|---|---|
| `stop=true`，n/len 不变 | **思考中**（旧消息仍是最后一条，别误判完成） |
| `stop=false` + `len>300` + 连续两次 len 相同 + n 已增长 | **完成** |
| `stop=false` + len≤4 或 0 持续数分钟 | **静默丢信**，走 §2 恢复 |

轮询循环（设最大轮数防挂死）：

```bash
for i in $(seq 1 60); do
  R=$("$B" --headed js "<状态表达式>" 2>/dev/null)
  echo "$R" | grep -q '"stop":false' && echo "$R" | grep -qv '"len":4' && echo "$R" | grep -qv '"len":0' && { echo "DONE $R"; break; }
  sleep 10
done
```

更稳：先记发出前的 `len` 基线，完成条件 = stop=false 且 len>基线 且连续两轮相同。

## 4. 回答抽取存档

⚠️ **`--out` 有路径白名单**，只允许 temp 或 browse dist 目录，写别处直接报错：

```
Path must be within: C:\Users\ADMINI~1\AppData\Local\Temp,
                     ~\.claude\skills\gstack\browse\dist
```

**正确姿势：先落 temp，再 cp 到目标目录。**

```bash
TMP="C:\\Users\\ADMINI~1\\AppData\\Local\\Temp"
OUT="<USER_HOME>/Desktop/xiaosu/photos"
mkdir -p "$OUT"

"$B" --headed js "[...document.querySelectorAll('[data-message-author-role=\"assistant\"]')].slice(-1).map(e=>e.innerText).join('\n\n---\n\n')" --out "$TMP/gpt_r1.txt"
cp "$TMP/gpt_r1.txt" "$OUT/gpt_r1_主题.txt"
```

- 取最后 N 条把 `slice(-1)` 改 `slice(-N)`。
- 落盘后**必须 `ls -la` 看大小 + 抽查内容**，空文件 = 抽取失败。
- 命名约定：`gpt_r1_主题.txt`、`gpt_r2_主题.txt`…放任务专属目录。

## 5. 生图：收割与下载

**完成判定（fragment 收割法）**：

```bash
"$B" --headed js "(()=>{const m={};[...document.querySelectorAll('img')].forEach(i=>{const x=(i.src||'').match(/id=(file_[a-f0-9]+)/);if(x){const f=x[1].slice(13,19);m[f]=(m[f]||0)+1}});return JSON.stringify(m)})()"
```

- 开工前先跑一次记下**已知 fragment**（旧图）；之后出现 **≥3 张 img 共享同一个未知新 fragment** = 本轮生成完成。
- 生图排队可能要几分钟，轮询同 §3 节奏。

**下载（同步 canvas，唯一可靠配方）**：

```bash
TMP="C:\\Users\\ADMINI~1\\AppData\\Local\\Temp"
"$B" --headed js "const frag='新fragment';const img=[...document.querySelectorAll('img')].find(x=>(x.src||'').includes(frag));const c=document.createElement('canvas');c.width=img.naturalWidth;c.height=img.naturalHeight;c.getContext('2d').drawImage(img,0,0);c.toDataURL('image/png')" --out "$TMP/shot1.png"
cp "$TMP/shot1.png" "$OUT/交付图.png"
```

- ⚠️ **browse js 不 await promise**：fetch/blob/async IIFE 全部静默失败产出 0 字节文件，**必须用同步 canvas**。
- 下载完 `ls -la` 验字节数，**再目检图片内容**（是否熔脸/文不对题）。
- 交付前把图发给用户，**不能只报一个本地路径就当交付**。

### 5.0 ⚠️ 检测方法已更新：新版生图用 blob URL（2026-10-08 实测）

**旧方法失效**：`id=file_<hex>` 的 fragment 收割法**已不适用**。新版 ChatGPT 生图返回
`blob:https://chatgpt.com/<uuid>`，`--out` 的 fragment 匹配永远为空 → **会误判成"没出图"**。

**实测代价**：用旧方法轮询 12 轮全部报 0，白等 4 分钟、重复发消息 2 次、还点了"重试"，
**实际图早就出来了**。

**新检测式（唯一可靠）**：

```bash
"$B" --headed js "(()=>{const b=[...document.querySelectorAll('img')].filter(x=>x.naturalWidth>800&&(x.src||'').startsWith('blob:'));return JSON.stringify({blob:b.length,sizes:b.map(x=>x.naturalWidth+'x'+x.naturalHeight)})})()"
```

- `blob` 从 0 变 ≥1 = **出图了**。典型尺寸 `1448x1086`。
- 一般 **3~4 轮（每轮 20s）** 就出，不用轮 12 轮。

**下载（同步 canvas，注意不能用旧 fragment）**：

```bash
"$B" --headed js "(()=>{const im=[...document.querySelectorAll('img')].filter(i=>i.naturalWidth>800&&(i.src||'').startsWith('blob:'));const img=im[im.length-1];const c=document.createElement('canvas');c.width=img.naturalWidth;c.height=img.naturalHeight;const g=c.getContext('2d');g.fillStyle='#fff';g.fillRect(0,0,c.width,c.height);g.drawImage(img,0,0);return c.toDataURL('image/png')})()" --out "$TMP/out.png"
```

- **必须 `fillStyle='#fff'` 先铺白底**，否则 PNG 透明区变黑。
- `--out` 白名单含 `hermes/cache/scratch`（不只有 `%TEMP%`）。
- 文件 2.5~4 MB 正常。

**另一个坑：页面文字会显示「编辑」二字**

新版界面在生成中的回复正文常只显示「**编辑**」（图片工具的占位标签），**这不是错误提示**，
不代表编辑误判。看到「编辑」+ blob 图已出现 = **正常完成**。
只有「图像生成失败 / 重试」才是真失败。

### 5.0.1 ⛔ 下载时机：必须「比 baseline 多」才算新图（连踩两次）

1. **发消息前**先记录当前 blob 数 `base`。
2. 轮询时**只有 `n > base` 才算新图出好**。
3. **绝不能只看「页面上有没有 blob 图」**——上一张图仍挂在页面上，会立刻返回旧的，
   下载到上一张（实测两次下载到同一张图，**MD5 完全相同**）。

**注意**：页面重渲染/滚动会让旧 blob 被清掉，`n` 可能**回落**（如 base=8 变 6）。
稳妥做法：配合 §2 的独特子串验真，再取 `im[im.length-1]`；下载后**比对 MD5** 确认不是旧图。

### 5.0.2 上传参考图（图上图 / 改图必用）

browse 有 `upload` 命令：

```bash
# 选择器要用 accept 限定（页面上有多个 file input）
"$B" --headed upload "input[type=file][accept='image/*']" "/path/ref.jpg"
# 输出 Uploaded: ref.jpg (123456B) 即成功；随后页面上会多出该图的 blob 缩略图
```

- 上传后**先等 4~6 秒**再发文字，让附件挂上去。
- **大图先压缩**（4MB PNG 压到 ~500KB JPG 即可，`PIL thumbnail((1100,1100))`）。
- 一次可传多张（逐张 upload），**引用时用「图1/图2」编号**，GPT 能对上。

### 5.1 编辑误判：长 prompt 生图最常见的失败（2026-09-30 实测）

**症状**：GPT 不报错、界面显示「回答已完成」，但**一张图都没出**。追问才看到它说：

> *"I couldn't generate this because the image tool incorrectly treated the request as an edit requiring a source image."*
> *"the image tool is misreading this request as an edit and is requiring a source image"*

**根因**：会话里有过图片（或 prompt 太长/含"参考/保持一致/同系列"等措辞）时，生图工具会**误判为图生图编辑**，然后因找不到源图而静默失败。

**判据**：`stop:false` 且 img 长时间不增长，或 `stop:true` 但 img 数为 0 —— 两者都要**读页面正文确认**，别只看状态位。

**解法（按成功率排序）**：

1. **追问一句**（最有效，实测救回 4/10 张）：
   ```bash
   "$B" --headed js "(()=>{const el=document.querySelector('div[contenteditable=\"true\"]');el.focus();document.execCommand('insertText',false,'Generate this from scratch now.');return 'ok'})()"
   "$B" --headed press Enter
   ```
   GPT 自己也会在回复里主动建议这么做（原文：*"Please resend the same request in a new message (even just 'Generate this from scratch now')"*）。
2. **开新对话重发**（次有效）：`goto https://chatgpt.com/?model=auto`，等 14s，重发。
3. **点界面上的「重试」按钮**：用 JS 找 `button` 且 text 匹配 `/^重试$/` 后 `.click()`。**成功率一般**，常再次卡住。

**预防**：
- 每条生图需求**开新对话**发，别在同一对话连发多张（第 2 张起失败率陡增）。
- prompt 开头显式写 *"Draw a completely new picture. There is no source image in this conversation and nothing to edit."*
- 别用 `参考`、`保持一致`、`同系列` 这类词，容易被判为编辑。

**其它相关注意**：
- 生图**可能在「最后微调 90%」停留几分钟**，此时 `stop:false` 且 img=0 —— **不是失败，别重发**。等够 5–7 轮。
- 新 DOM 输入框是 `div[contenteditable="true"]`（ProseMirror），`#prompt-textarea` 已失效。
- `browse` 的按键命令是 **`press`**，不是 `key`。
- 用 `js` + `document.execCommand('insertText', ...)` 填内容比 `fill` 更适合长文，中文用 base64 中转避免 shell 转义。
- 下载用**同步 canvas**（见 §5），取 `[...document.querySelectorAll('img')].filter(i=>i.naturalWidth>=900)` 的最后一张最稳。

### 5.2 生图任务通用开头

每条生图 prompt 开头加一句，避免歧义：

> `Draw a completely new picture. No reference image needed, nothing to edit.`
> （若是改图，则改为：`这是改图任务，基于我上传的图直接修改。`）

## 6. 已踩坑速查

| 坑 | 解法 |
|---|---|
| bash 双引号内联 PowerShell 吞 `$var` | heredoc 加引号定界符写 .ps1：`cat > x.ps1 <<'EOF'` |
| bash 处理中文文件名乱码 | 目录名用 ASCII；中文名只经 Write 工具或 PowerShell 操作 |
| js 里复杂 `case`/嵌套引号报 `unexpected EOF` | 拆简单表达式；避免 case |
| user 消息计数对不上 | 虚拟化只挂载最后 ~2 条，用子串 hit 判断 |
| 以为发成功了其实没发 | 一切以 reload 后的子串检查为准 |
| daemon 报 config mismatch | 先确认 state file 指向对不对；别直接 `disconnect` |
| `ProcessSingleton ... profile is already in use` | 有 headed 浏览器占着 profile。先 `status` 复用；重启前先杀掉**只占这个 profile** 的孤儿 chrome |
| state file / 端口找不到 | **state file 按 cwd 发现**，换目录就丢。一律显式钉死 `BROWSE_STATE_FILE` |
| 同机有别的 agent 在跑 | 先只读探活（status/url/tabs）确认归属；杀进程前必须问用户 |
| `--out` 报 `Path must be within` | 先落 `%TEMP%`，再 `cp` 到目标目录 |
| 生图 fragment 找不到 | 先确认 `stop:false` 且 img 数增长；fragment 取自 `img.src` 的 `id=file_<hex>`，`slice(13,19)` 取 6 位 |
| **生图「回答已完成」但没出图** | **编辑误判**（长 prompt / 会话有旧图）。读正文确认，然后追问 `Generate this from scratch now.` —— 详见 §5.1 |
| 生图卡在「最后微调 90%」 | **不是失败**，等够 5–7 轮别重发 |
| 同一对话连发多张图 | 第 2 张起失败率陡增。**每条需求开新对话**发 |
| 按键命令 `key` 不存在 | 用 `press`；可用命令列表见 `browse` 无参运行时的输出 |
| `#prompt-textarea` 找不到 | 新 DOM 是 `div[contenteditable="true"]`（ProseMirror） |
| daemon 刚重启后 `#prompt-textarea` 不可见 | **不是浏览器坏了**，是停在欢迎页。补一步 `goto https://chatgpt.com/` 即可 |

### 6.1 daemon 生命周期与自愈（2026-09-22 沉淀，部分已被 09-24 修正——先读 §0.5）

> ⚠️ **09-24 更正**：本节"后台脚本一次跑完、任务内 daemon 全程稳定"的经验是**假象**——当时有 `browse-daemon.ps1` 保活脚本在背后 60s 托底（daemon 死了它秒拉新的）。保活已禁用（见 §0.5），此经验不再成立：`run_in_background` 的 shell 会失联 daemon，后台任务里的 browse 命令得到的只是"命令成功、结果全空"。现在的正确姿势 = **孤儿 daemon + 前台单条调用 ≤120s 完成完整动作**，跨调用靠孤儿进程续命。

- ~~daemon 生命周期 = 启动它的那次 Bash 调用/后台任务~~ → **修正**：CLI spawn 的 daemon 成孤儿后可跨调用存活。
- ~~自愈包装器（每次调用先 status，死了 kill+重启）~~ → **修正**：自动 kill+重启违反 §0.5 铁律 3（静默重开会弹窗骚扰用户）。daemon 死了先请示。
- daemon 重启后**显式 goto 线程 URL（`/c/<id>`）比点 sidebar 稳**；重启后首次加载慢，composer 轮询要给 120s。
- **僵尸实例是最大污染源**：同脚本多实例并发会互相往同一线程发消息。启动新实例前必须查残留并树杀。
- **probe 落地检查的判读纪律**：daemon 死亡瞬间做的子串检查返回 false 不可信；`true` 可信，`false` 要先确认页面在 `/c/` 且加载完再判丢信。

### 6.2 内存与生图杂症

- **大模型（ComfyUI 等）吃内存会挤死 chromium**（实测 43.7GB python 把 daemon 压到每分钟死一次）。解法：`curl --noproxy "*" -X POST http://127.0.0.1:8188/free -d '{"unload_models":true,"free_memory":true}'`；跑视频时再让 ComfyUI 冷加载。
- GPT 生图偶发服务端报错文本"Something went wrong while generating your image"——**报错后可能仍带着成功生成的图**。照常查 blob 列表+下载目检。
- H3/重活与浏览器操作分时跑：先收完 GPT 图，再提交视频生成。

## 6.5 角色一致性：生成卡通角色系列的打法（2026-10-08 实测）

做一批同角色配图（如绘本角色）时的实测规律：

1. **不要追求严格三视图。** AI 画不好正交三视图（正面/侧面/背面的发束、口袋、鞋细节对不上）。
   实测迭代 3 版仍不一致。**除非要 3D 建模，否则不值得。**
2. **改成「表情设定图」**（同一角色 × 6 个表情/动作）。实测**一轮就通过**，且更实用 ——
   内容配图需要的是不同情绪，不是转面精度。
3. **「夸张」≠ 把尺寸参数推到极端。** 实测教训：把头身比推到 1.3、瞳孔缩到针尖、
   眼睛占脸 80% —— **参数全达标，但结果被评价为「傻子」**。
   夸张 = **放大情绪**（表情 + 身体动作），不是放大比例。
4. **自查表**（生成后逐项核对）：眼睛是否随表情变化？嘴巴是否有大有小？
   眉毛是否参与情绪？身体是否有动作？**是否只是同一姿势的多个副本？**
5. **定稿后再批量。** 角色基线未确认前批量出图 = 全部返工。
6. **提示词必须包含负向描述**：`Avoid: realistic proportions, small eyes, mature face,
   photorealistic, 3D render, natural pose` —— 对卡通化效果提升明显。
7. **迭代节奏**：每版都让 vision 打分并**逐项核对具体要求**，比盲目重试快得多。
   典型 3~5 版收敛；超过 6 版说明提示词的**结构**错了，该换思路而非调参数。

## 6.6 已有风格基准时：必须「改图」而非「照着重画」（2026-10-09 实测，代价极大）

**症状**：用户给一张风格基准图（如某角色的六宫格），要求「其他角色都用这个画风」。

**错误做法（我走了 6 版的弯路）**：用文字描述那张图的画法（「水彩+彩铅+低饱和+细长杏眼+暖褐线条…」），
然后让 GPT「用这个画法画一个新角色」。
→ **必然失败**。结果永远是「同色系的另一种画风」，用户会说「完全不是一种画风，放一起很违和」。

**根因**：**画风是图像的属性，不是文字的属性。** 材质词（水彩/彩铅/纸纹）能描述，
但**画法**（眼睛多大、占总脸多少、线条多细、脸型怎么收）**描述得再细也还原不了** ——
GPT 会把「细长杏眼」理解成「杏眼但很大」，把「半写实」理解成「绘本卡通加水彩滤镜」。

**正确做法**：
1. 把基准图 **`upload` 进对话**（见 5.0 上传法），让 GPT 先看图、复述画法
2. 明确说「**这是改图任务，不是重新创作**」
3. 逐项列出**改什么**（人物身份/服装/道具），并强调「**其余全部原样保留**：
   画法、线条、上色、纸张质感、构图、视角、每格姿态表情、背景」
4. 引导它照搬原图的**构图和分镜**，只换主体

**实测效果**：同一张基准图，文字描述重画 → 用户说「完全不像」；
改图法一次成功 → 画风、线条、眼型、头身比全部对上。

**推论（通用）**：只要用户提供了任何风格基准图，**第一选择永远是上传图片做改图**，
文字描述只作为补充说明「改哪些地方」。

## 7. 红线

- **绝不向 ChatGPT 发送服务器 IP、token、密钥、内网域名、客户敏感信息**——提示词只谈产品能力、公开行情与结构化需求。
- 每轮回答必须落盘存档，防止线程丢失后讨论成果蒸发。
- 工具不可达时如实挂起请求，不假装执行。

## 8. 多轮讨论协议（实测高效打法）

1. **R1 场景穷举** → R2 定方向/产品定义 → R3 数字（成本/定价/账）→ R4 执行（营销/GTM/排期）。每轮只解决一层。
2. 每条提示词结构：**上轮结论编号回顾（压缩几行）→ 本轮编号问题 → 明确输出格式（"用表格、给具体数字、不要泛泛而谈"）**。GPT 在被要求表格化+数字化时输出质量显著更高。
3. 节奏：发 → 30s 验证落地 → 轮询完成 → `--out` 存档 → 消化 → 结论压进下一轮提示词。
4. 需要"汇报素材"时用生图（§5），概念图足够则不浪费时间生成视频。
5. 收尾：综合全部轮次产出完整方案文档；GPT 引用的外部数据（法规/竞品价格）标注"需二次核实"。

---

## 9. Windows 排障（daemon 起不来时按节序查）

本机 browse 链路与上游假设**不同**，出问题按这四层顺序查。**每层都有实测证据，别跳步。**

### 9.0 一键启动

```bash
bash ~/.claude/skills/gstack/browse/start-browse.sh
# ✅ browse daemon 就绪 (PID xxx, 端口 xxx)
```

脚本做的事：绕开 `browse.exe` 用 node 起 `server-node.mjs` → 等 state file → 补 `configHash`。**幂等**。

### 9.1 第一层：`browse.exe` 在 Windows 上不能用

**症状**：任何 `browse` 命令报 `Server failed to start`，日志 `exitCode=2147483651`（`0xC0000003`）。

**根因**：`dist/browse.exe` 是 Bun 编译产物。Windows 上 Bun 驱动不了 Playwright Chromium（oven-sh/bun#4253）。gstack 自身设计是回头用 Node 跑 `server-node.mjs`——但 `browse.exe` 内部 launcher 用 `process.execPath`，在 Bun 二进制里指向 **Bun 自己**，子进程仍是 Bun，加载 `bun-polyfill.cjs` 时撞：

```
TypeError: Attempted to assign to readonly property.   (bun-polyfill.cjs:16)
```

Bun ≥1.3 的 `globalThis.Bun` 已只读。**死循环，必须绕开 `browse.exe`。**

**解法**：直接 node 起（`start-browse.sh` 已封装）：

```bash
BROWSE_STATE_FILE="<USER_HOME>/.gstack/browse-state.json" \
BROWSE_PARENT_PID=0 BROWSE_HEADED=1 \
CHROMIUM_PROFILE="<USER_HOME>/.gstack/browse-profile" \
  node "$HOME/.claude/skills/gstack/browse/dist/server-node.mjs"
```

### 9.2 第二层：`server-node.mjs` 的两处 Node 兼容 bug

上游是给 Bun 写的，用 node 跑踩两个坑。**已打补丁**（备份 `server-node.mjs.bak-hermes-*`）。**gstack 升级后需重打。**

**坑 A — 沙箱访问不到 chrome.exe**

```
ERROR:sandbox\policy\win\sandbox_win.cc:780] Sandbox cannot access executable ...\chrome.exe
```

`launchHeaded` 的 `launchArgs` 没带 `--no-sandbox`。**解法**：在 `launchHeaded` 的 `launchArgs` 里补 `"--no-sandbox"`。

⚠️ 有两处同名数组：`STEALTH_LAUNCH_ARGS`（**改这里不生效**）和 `launchHeaded` 里的 `launchArgs`（**真正生效**）。用 `grep -n "launchArgs = \["` 确认改对地方。

**坑 B — `browser?.process()` 不存在**

```
[browse] FATAL unhandled rejection: browser?.process is not a function
```

`resolveDisconnectCause` 调了 playwright `Browser` 上不存在的方法，server 启动后立刻退出。**解法**：

```js
// 改前
const proc = browser?.process();
// 改后
const proc = typeof browser?.process === "function" ? browser.process() : null;
```

### 9.3 第三层：profile 锁残留 / profile 损坏

**症状 A：`Lock file can not be created! Error code: 32`**

有**孤儿 chrome** 占着 profile（多半是之前失败的测试留下的）。诊断：

```bash
wmic process where "name='chrome.exe'" get ProcessId,CommandLine /format:csv 2>/dev/null | tr -d '\r' | grep "browse-profile" | awk -F, '{print $NF}'
```

杀掉列表里的 PID（**只杀带 `browse-profile` 的，别碰用户正版 Chrome**），确认归零后重起。

**症状 B：`launchPersistentContext ... Target page, context or browser has been closed`**

profile 本身损坏。**验证**（换新目录试）：

```bash
CHROMIUM_PROFILE="<USER_HOME>/AppData/Local/hermes/cache/scratch/prof_test" \
BROWSE_STATE_FILE=... BROWSE_HEADED=1 node server-node.mjs
# 起来了 → 就是旧 profile 坏了
```

**解法**：换独立 profile（`~/.gstack/browse-profile`）。**别再调 launch 参数**——实测 `--disable-gpu` / `--in-process-gpu` / `--use-angle` / `--disable-software-rasterizer` / UA / sandbox 组合**全都不是原因**，参数等价时换 profile 就好。

**GPU 崩溃是伴随现象**：dump 里 `vk_swiftshader.dll` + `vkCmdDrawIndexed` + `angle:SwANGLE` 是 Intel 集显下 SwiftShader 软件渲染崩溃，属噪音，不影响主进程。

### 9.4 第四层：迁移 ChatGPT 登录态到新 profile

⚠️ **Chrome 新版 Cookies 在 `Default/Network/Cookies`，不在 `Default/Cookies`。** 查错路径会误判成"登录态丢了"。

**三个条件必须同时满足，缺一就白搬**：

1. **搬 Cookies**：`<profile>/Default/Network/Cookies`
2. **搬加密密钥**：`<profile>/Local State` 的 `os_crypt` 段。不同 profile 的 DPAPI `encrypted_key` **不同**，只搬 Cookies 解不开（表现：cookie 数正常但登录态是访客）。
3. **搬运时所有 chromium 进程必须已停**。否则浏览器退出时把内存里的空数据写回，**覆盖迁移结果**（实测：搬完 102 条 → 起浏览器 → 退出后剩 8 条）。

**正确顺序**：

```bash
# 1) 停 daemon + 所有用目标 profile 的 chromium，确认归零
# 2) cp "$OLD/Default/Network/Cookies" "$NEW/Default/Network/Cookies"
# 3) python 改 JSON，只替换 os_crypt 段：n["os_crypt"] = o["os_crypt"]
# 4) 起 daemon → goto chatgpt.com → 验 cookie 数 + session-token
```

**验证登录态成功的硬指标**：

```bash
"$B" --headed js "document.title"                      # ChatGPT
"$B" --headed is visible "#prompt-textarea"            # true（访客态 false）
"$B" --headed cookies | grep -c '"name"'               # 80+ 条
"$B" --headed cookies | grep -o "next-auth.session-token[^\"]*"   # 必须命中
```

**还是访客态**：别反复重试。先确认 2 和 3 都做了——绝大多数失败是漏搬 `Local State` 的 `os_crypt`。

### 9.5 别踩的坑

| 坑 | 后果 |
|---|---|
| 用 `browse.exe` 起 daemon | 必崩（§9.1） |
| 改 `STEALTH_LAUNCH_ARGS` 想加参数 | 不生效，真正生效的是 `launchHeaded` 的 `launchArgs` |
| 搬 Cookies 不搬 `Local State` | 解不开，表现为"登录态丢了" |
| 有 chromium 在跑时搬 Cookies | 被退出回写覆盖 |
| 查 `Default/Cookies` 判登录态 | 路径错了，误判丢失 |
| 直接 `browse disconnect` 解 config mismatch | 会杀同机 agent 在用的 daemon |
| 同机多 agent 共用 profile 时抢起 daemon | 单例锁冲突。先只读探活，动手前问用户 |
| 启用 `browse-daemon.ps1` 保活循环 | **无限弹窗事故**（杀名单过宽误伤别的会话）。保持禁用 |
| daemon 停着就不管了 | 云端桥依赖它。停之前先确认没有远端任务在跑 |

### 9.6 本机现状

- 启动：`bash ~/.claude/skills/gstack/browse/start-browse.sh`（幂等，按需单次）
- state file：`<USER_HOME>/.gstack/browse-state.json`
- profile：`<USER_HOME>/.gstack/browse-profile`（已迁入 ChatGPT 登录态，账号 <ACCOUNT>/Plus）
- 旧 profile：`<USER_HOME>/.gstack/chromium-profile`（已损坏，保留作 Cookie 恢复源）
- 补丁备份：`server-node.mjs.bak-hermes-*`
- 数据备份：`hermes/cache/scratch/profile-backup-*`

---

## 10. 云端 Agent 桥接（生产部署，2026-09-24 实测跑通）

云端 OpenClaw 通过反向 SSH 隧道驱动本机浏览器生图。

**链路**：

```
云服务器 <SERVER_IP>
   └─ <SERVER_PATH>/ws-browse（封装脚本）
        └─ ssh -p 22022  ← 反向隧道
             └─ 本机 Git bash
                  └─ browse daemon（headed，桌面会话）
```

### 10.1 隧道（本机侧发起）

```bash
ssh -N -R 127.0.0.1:22022:127.0.0.1:22 \
  -i "C:/Windows/System32/config/systemprofile/.ssh/<TUNNEL_KEY>" \
  -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -o ExitOnForwardFailure=yes \
  root@<SERVER_IP>
```

**注意**：用 `C:\Windows\System32\config\systemprofile\.ssh\<TUNNEL_KEY>`（**不是** `~/ssh-tunnel/<TUNNEL_KEY>`，那把的权限会被 sshd 拒绝）。

**云端自检**：`ss -tlnp | grep 22022` 应有 LISTEN。

### 10.2 回连密钥（服务器 → 本机）

服务器需要一把能登本机的私钥，路径 `~/.ssh/<WS_KEY>`。**丢了就重建**：

```bash
# 本机生成
ssh-keygen -t ed25519 -f /path/<WS_KEY> -N "" -C "xiaosu-ws-bridge"
# 公钥加进本机授权（两个文件都要加！）
#   C:\ProgramData\ssh\administrators_authorized_keys
#   <USER_HOME>\.ssh\authorized_keys
# 私钥 scp 到服务器 ~/.ssh/<WS_KEY>，chmod 600
```

⚠️ 本机 sshd 的 `AuthorizedKeysFile` 是 `.ssh/authorized_keys`，**只加 `administrators_authorized_keys` 不够**——两个都加。

### 10.3 `ws-browse` 用法（云端侧）

```bash
ws-browse ensure                          # 预检 + 自动拉起 daemon + 导航到 ChatGPT
ws-browse status | url | goto <url>       # 任意 browse 子命令透传
ws-browse is visible '#prompt-textarea'   # 登录态硬指标
ws-browse download <fragment> <输出路径>  # 取回图片（自动 base64 解码）
```

**`ensure` 是关键**：它会自动处理「daemon 不在 → 拉起」和「停在欢迎页 → 导航到 ChatGPT」。**云端每次开工前先跑一次 `ensure`**，就不会再出现「`#prompt-textarea` 是 false 就以为浏览器坏了」的误判。

### 10.4 三条硬约束

1. **`--out` 过隧道必被拒**（隧道不开 write scope）。生图下载**必须用 `ws-browse download`**，别照抄 §5 的 `--out` 命令。
2. **明文参数会被多层转义吃掉**。中文/复杂参数用 base64 中转（`ws-browse` 内部已处理；手写 ssh 命令时注意）。
3. **隧道和 daemon 都不是自启的**。工作站重启后全断。保活必须"按需单次启动"，**不要恢复 `browse-daemon.ps1` 的循环**（详见 §11 事故记录）。

### 10.5 云端完整生图流程

```bash
# 1) 预检
ws-browse ensure

# 2) 发需求（中文用 base64 中转）
B64=$(printf '%s' "$PROMPT" | base64 -w0)
ssh -p 22022 -i ~/.ssh/<WS_KEY> Administrator@127.0.0.1 \
  "P=\$(echo $B64 | base64 -d); ws-browse fill '#prompt-textarea' \"\$P\" && ws-browse press Enter"

# 3) 30s 后验落地（独特子串）
ws-browse js "[...document.querySelectorAll('[data-message-author-role=\"user\"]')].map(e=>e.innerText).join('').includes('独特标记')"

# 4) 轮询 fragment 直到 ≥3 张共享
ws-browse js "(()=>{const m={};[...document.querySelectorAll('img')].forEach(i=>{const x=(i.src||'').match(/id=(file_[a-f0-9]+)/);if(x){const f=x[1].slice(13,19);m[f]=(m[f]||0)+1}});return JSON.stringify(m)})()"

# 5) 下载（自动 base64 回传 + 解码）
ws-browse download <fragment> /root/output.png
```

**实测**：中秋海报（3.4MB）、极简验证图（1.15MB）均走此流程成功产出并目检通过。

---

## 11. 事故记录（防止重蹈覆辙）

### 无限弹窗（2026-09-24）

**现象**：有头浏览器全天反复弹出，严重干扰用户使用电脑（用户多次投诉）。

**根因**：`ssh-tunnel/browse-daemon.ps1` 的 `while($true)` 循环——每 60 秒查一次 daemon，不健康就**清杀整个 browse 栈**（`server-node.mjs` + `chromium-profile` chrome）再重拉 headed 窗口。**杀名单过宽，误伤其他会话正在用的 daemon。**

**处置**：循环进程已停，Startup 项改名 `browse-daemon.cmd.disabled`。

**铁律**：**不要恢复这个循环。** daemon 一律按需单次启动，且必须先征得用户同意。

### 登录态误判（2026-09-24）

**现象**：以为 ChatGPT 登录态丢了。

**根因**：查的是 `Default/Cookies`，实际路径是 `Default/Network/Cookies`。登录态一直好好的。

**教训**：判断登录态先看 `#prompt-textarea` 是否可见，别靠文件路径猜测。

### 「浏览器坏了」误判（2026-09-24）

**现象**：云端报「headed Chromium 秒退，需要重启工作站」。

**根因**：daemon 刚重启停在欢迎页（`http://127.0.0.1:<port>/welcome`），`#prompt-textarea` 自然不可见。**云端没做 `goto` 导航**，误判成浏览器故障。

**教训**：`ws-browse ensure` 已自动补这一步。看到 `false` 先查 URL，别急着重启工作站。
