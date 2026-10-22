---
layout: post
title: "AI 编程工具到底传了你多少代码？从智谱 ZCode 风波讲起"
date: 2026-09-21 07:14:51 +0800
categories: 公众号
tags: [Kotlin, 公众号]
cover: /assets/images/zcode-upload/cover.png
---

![AI 编程工具到底传了你多少代码？从智谱 ZCode 风波讲起](/assets/images/zcode-upload/cover.png)

## 从一次磁盘清理开始

技术博主 ferstar 在整理磁盘时，发现 `~/.zcode` 这个目录占得有点多——超过 700MB。

他往下翻。`cli/` 257MB，`computer-use/` 约 130MB，都还算正常。真正的大头是 `v2/checkpoints/`，303MB。

里面躺着一个 313MB 的 `.enc` 文件。

打不开。但旁边那份状态元数据是明文的，写着这个工作区是一个商业项目，类型标记为「全量快照」，仓库总量约 10GB，剔掉各种依赖后剩下的 345MB 是项目核心资产。

也就是说，这个他打不开的文件里，装着他的整个项目。

这是 9 月 18 日的事。当天他发布了对 ZCode 的逆向分析，随后智谱致歉，承认问题源于一项默认开启的「代码库索引」功能。

但这件事真正值得写的地方，不在热搜那部分。

## 他把上传链路整个拆开了

日志里没直接写上传地址，ferstar 是拆开客户端文件，把整条链路还原出来的：

1. 客户端先向 `zcode.z.ai` 请求上传凭证
2. 服务端返回 OSS 表单签名、动态对象 Key、大小限制，以及本轮临时下发的一把 RSA 公钥
3. 客户端在本地打包、流式加密，生成 `.enc` 文件
4. **绕过智谱自己的业务服务器**，以 POST 表单把加密文件直传到阿里云 OSS
5. OSS 再回调智谱后端做登记

加密方案是标准信封加密：内容用临时对称密钥走 AES-256-CTR，对称密钥再用服务端下发的公钥做 RSA-OAEP-SHA256 包裹。

关键在于——**私钥全程只在云端**。

ferstar 试了本机所有私钥，解不开。这个设计意味着：用户自己打不开这个文件，ZCode 客户端也解不开，只有智谱后端能解密。

他自己也说得很克制：如果这个功能是为断点恢复或跨设备同步设计的，密钥应该留在本地。只有服务端能解开的加密包，那它至少能保证一件事——只有服务端能单方面查看里面的内容。

## 那个包里到底有什么

密文打不开，但客户端打包时留下的明文 Manifest 是可以读的。42411 个文件的清单，拆出来是这样：

| 目录 | 体积 | 占比 | 是什么 |
|---|---|---|---|
| `.git/lfs/` | 196.1 MB | 56.8% | 历史上拉过的全部大文件与二进制资产 |
| `.git/objects/` | 102.2 MB | 29.6% | 完整的 commit / tree / blob 对象库 |
| `.git/logs/` | 0.6 MB | — | 本地分支操作、未推送的记录 |
| 源码与文档 | 46.2 MB | — | 当前工作树 |
| `.git` 合计 | — | 86.6% | |

这里有个很多新闻没讲透的点：**云端拿到的不是你的当前代码，是你的代码从出生到现在的全部经历。**

包括已经被覆盖或删除的敏感配置、历史上提交过又删掉的老密钥、还没推送上去的分支名、以及 `.git/config` 里你们公司内部 GitLab 的域名和仓库路径。

还有个更隐蔽的问题。ferstar 对照代码发现，ZCode 的文件筛选是按顺序判定的，而「历史记录目录」的放行**排在密钥过滤和体积限制之前**。

结果就是：针对 `.pem`、`.key` 这类密码文件的过滤、以及 1MB 的体积上限，**对 `.git` 目录下的任何内容都不生效**。

所以一个上百兆的包能整个带走，里面任何历史上提交过又删掉的密码和密钥，也原样跟着走。

## 最要命的是：开关是假的

界面上一眼能看到的两个相关选项，ferstar 逐个对照代码逻辑确认了实际作用：

- 「优化体验」 → 只控制数据是否用于模型训练
- 「仓库快照索引」 → 只控制服务端收到数据后是否建立检索目录

**两个都关掉，本地的打包和上传照常运行。**

负责快照和上传的组件在软件启动时就被无条件实例化，没有任何用户偏好判断，唯一的前提是你处于登录状态。

触发点有两个：每次发 Prompt 之前的 `captureBeforePrompt`，以及任务结束时的 `repo-wiki-update`。他在一个活跃会话的日志里，观察到最多 62 次捕获记录。

他还试过手动删那个 313MB 的待发送文件。**半小时后，ZCode 重新生成了一份新的。** 上传不成就重试。

而智谱的隐私政策里写的是收集「对话中提交的文本、文件和代码」——**没提过会静默打包上传整个工作区和完整 Git 历史**。官方文档、FAQ、更新日志，同样一个字都没有。

（社区里给出的临时办法是清空并锁定 `~/.zcode/v2/checkpoints/` 目录，macOS 用 `chflags uchg`，Linux 用 `sudo chattr +i`。代价是检查点回滚和时间线界面失效，对话、补全、工具执行不受影响。后面我会把命令整理进去。）

## 官方回应之后，事情没结束

智谱 9 月 18 日致歉，说功能本意是本地生成仓库索引，支持历史版本回退和 Repo Wiki；Wiki 页面在云端生成后数据立即销毁、不会保存；默认开启导致部分用户受影响，已修复。同时承诺近期开源 ZCode 代码库、引入第三方评估人员审查、给全体用户补偿一次周额度重置。

四天后，企业侧的反应比个人用户更硬。

用户方太原承明科技 19 日晚间公开发函，向北京智谱华章提出 12 项答复要求。他们的独立取证结果：

- 自己这边被上传的是 **6 个工作区、34549 个文件、约 425MB**，含源代码、版本控制历史、配置文件，以及数据库口令、云服务凭证和员工个人信息
- 上传是自动触发、批量发生的，远超隐私政策载明的收集范围
- ZCode 客户端 9 月 16 日已更新到 3.12.3，但在智谱 9 月 18 日公开致歉当天的凌晨，仍检测到上传行为
- 客户端的网络请求指向**新加坡主体**，而服务协议签约主体是北京智谱华章——要求说明责任主体归属，以及数据是否出境传输或境外存储

他们的诉求包括：立即停止处理、彻底删除已上传数据及其衍生数据/缓存/备份、提供完整的数据处理情况清单、书面说明数据去向。

据红星资本局报道，智谱对媒体采访的回应是四个字：「不实消息」。

## 这不是孤例，而且都不是监管查出来的

同一时间，别的工具也被翻了出来。

独立安全研究者 cereblab 用抓包证明，xAI 的编程 Agent Grok Build 会把用户的整个项目打包上传到谷歌云存储——包括用户明确告诉 AI 不要读取的文件，以及未经脱敏的密码。

更早一些，Claude Code 被发现会在用户不知情的情况下回传位置和身份信息。Anthropic 的工程师事后确认，那是一次**有意的实验**。

把这些放在一起看，会发现一个让人不太舒服的规律：

**这些事情没有一件是被监管或安全审计发现的，全部来自社区里的个人开发者。**

而目前行业给 Agent 搭的安全框架——模型对齐、提示词注入防护、工具权限边界——没有一条规则是用来约束厂商自己行为的。

过去两年，涉及 Agent 安全的讨论重心一直在「模型会不会失控」「提示词会不会被投毒」「外部攻击者怎么进来的」。厂商自身的数据行为，很少被当成一级风险来讨论。

现在这个前提被打破了。

所以结论其实很朴素：**在这种环境下，指望别人替你把关是不现实的，你得自己会查。**

下面是我整理的一套能落地的自查方法，从个人到企业内网，分四步。

## 第一步：先搞清楚你在用本地推理还是云端推理

这一步之所以重要，是因为它决定了后面所有检查的形态。

纯本地推理有个很硬的物理特征：它不太需要出网。模型就在你硬盘上，算力在你机器上。

反过来，只要是云端推理，你的 prompt（以及被工具塞进上下文的文件内容）必然要经过一条网络请求出去。

先看模型文件是否真的存在：

```bash
# Ollama 的模型 blob 目录，真跑本地模型这里一定有东西
du -sh ~/.ollama/models/blobs 2>/dev/null

# 常见的本地模型文件格式
find ~ -maxdepth 5 \( -name "*.gguf" -o -name "*.safetensors" \) -size +100M 2>/dev/null | head
```

再看推理时这个进程在干什么：

```bash
PID=$(pgrep -f "ollama|llama-server|lmstudio" | head -1)

# 它有没有在跑网络连接？
sudo lsof -i -P -n -a -p "$PID"

# 它的 CPU / 内存占用（本地推理会吃掉等于模型体积的内存）
top -pid "$PID" -l 1 | head -12
```

判断口径很简单：

- **有出网连接 + 内存占用很小 + CPU 不忙** → 云端推理，你的内容出去了
- **没有出网连接 + 内存占用约等于模型体积 + GPU/CPU 明显忙** → 本地推理

顺便说，这一条也能反过来验伪。有些工具在宣传页写「本地运行」，你按上面的命令一看，进程正连着 `api.xxx.com`——那就不用再猜了。

## 第二步：抓包，直接看请求体

这是最直接的手段，也是 cereblab 证明 Grok Build 用的方法。

装 mitmproxy：

```bash
brew install mitmproxy          # macOS
# 或者 sudo apt install mitmproxy
```

写一个只记录、不改写的插件，专门盯请求体大小和开头内容：

```python
# ~/ai_traffic_dump.py
from mitmproxy import http
import json, os

LOG = os.path.expanduser("~/ai_traffic.log")

# 只关心这些方向的流量，其他的不记
WATCH = ("api.", "openai", "anthropic", "z.ai", "aliyun",
         "googleapis", "zhipu", "bigmodel")

def request(flow: http.HTTPFlow) -> None:
    host = flow.request.pretty_host
    if not any(k in host for k in WATCH):
        return

    body = flow.request.get_text() or ""
    rec = {
        "host": host,
        "method": flow.request.method,
        "url": flow.request.pretty_url[:300],
        "ctype": flow.request.headers.get("content-type", ""),
        "body_bytes": len(body),
        "body_head": body[:1200],
    }
    with open(LOG, "a", encoding="utf-8") as f:
        f.write(json.dumps(rec, ensure_ascii=False) + "\n")
```

启动，并把工具流量导过来：

```bash
mitmproxy -p 8080 -s ~/ai_traffic_dump.py
```

另一个终端里：

```bash
export HTTPS_PROXY=http://127.0.0.1:8080
export HTTP_PROXY=http://127.0.0.1:8080
```

首次使用要信任 mitmproxy 的根证书（`~/.mitmproxy/mitmproxy-ca-cert.pem`，导入系统钥匙串并设为始终信任），否则 TLS 握手会失败。

用一会儿之后，按请求体大小排个序——谁传得多，一眼就能看出来：

```bash
python3 - <<'PY'
import json, os
rows = [json.loads(l) for l in open(os.path.expanduser("~/ai_traffic.log"))]
rows.sort(key=lambda r: -r["body_bytes"])
print(f"{'体积':>12}  {'方法':<6} URL")
for r in rows[:25]:
    kb = r["body_bytes"] / 1024
    print(f"{kb:>10.1f}KB  {r['method']:<6} {r['url'][:88]}")
PY
```

一个正常的对话请求，请求体通常是几 KB 到几十 KB。如果你看到某条请求体是几 MB 甚至几十 MB，而你这轮只是让它改了一行代码——那就值得打开 `body_head` 看一眼了。

顺带一个容易忽略的点：很多客户端**不认 `HTTPS_PROXY` 环境变量**——尤其 Electron/Node 写的工具和自带网络栈的客户端。这类情况得用 mitmproxy 的透明代理模式，或者在系统层做 TUN 接管。

## 第三步：抓不到怎么办——退一步看流量体量

有些客户端会做证书固定（certificate pinning），你中间人塞证书会直接握手失败。这时候别在解密上死磕，不解密也能看出很多东西。

macOS 上按进程看实时网络流量：

```bash
# 实时看每个进程的收发字节数
sudo nettop -m tcp -l 1 -t wifi | head -25

# 只看某个工具
sudo nettop -P -J bytes_in,bytes_out -l 1 | grep -i -E "zcode|codex|cursor"
```

Linux 上：

```bash
# 按进程看连接与流量
ss -tp state established
# 或者持续观察
sudo nethogs
```

**判据还是那一条：一次「只改了一行代码」的交互，把一个几百 KB 的请求推出去，和推出去几百 MB，是完全不同量级的事。** 后者基本可以确定是在打包上传。

另外看一眼那个工具在本地攒了什么：

```bash
# 个人侧最有效的一招：找异常大的隐藏目录
du -sh ~/.zcode ~/.cursor ~/.codex ~/.continue 2>/dev/null | sort -hr

# 找体积异常的加密包 / 快照
find ~ -maxdepth 4 -type f \( -name "*.enc" -o -name "*checkpoint*" \) -size +50M 2>/dev/null
```

ferstar 就是从 `du -sh` 这一步开始的。如果某个工具在本地留了一堆你从没见过的、打不开的大文件，那它多半有一个你没被告知的上传链路。

如果你确认了这类目录，可以按社区的办法先把它冻上：

```bash
# macOS：给目录加不可变标记
chflags -R uchg ~/.zcode/v2/checkpoints

# Linux
sudo chattr -R +i ~/.zcode/v2/checkpoints

# 解除
chflags -R nouchg ~/.zcode/v2/checkpoints
sudo chattr -R -i ~/.zcode/v2/checkpoints
```

代价要提前说清楚：**检查点回滚和时间线界面会失效**，但对话、补全、工具执行不受影响。这是个应急手段，不是长期方案。

## 第四步：企业内网的兜底做法

个人靠抓包已经很吃力了——`nettop` 一次只能看一台机器。真正需要管控的地方是团队和公司。

这时候的正确姿势不是封端口（AI 工具都走 443，一封全死），而是**在出口做域名白名单**。

一个能直接跑的观察版代理：

```python
# /opt/egress/egress_proxy.py —— 先只记录，不拦截
import fnmatch, time
from mitmproxy import http

ALLOW = [
    # 代码托管
    "github.com", "*.github.com", "*.gitlab.com",
    # 包管理
    "registry.npmjs.org", "pypi.org", "*.pypi.org",
    "repo.maven.apache.org", "*.gradle.org",
    # 已批准的模型服务（按你们实际用的加）
    "api.deepseek.com", "open.bigmodel.cn",
]

LOG = "/var/log/ai-egress.log"

def _allowed(host: str) -> bool:
    return any(fnmatch.fnmatch(host, p) for p in ALLOW)

class Egress:
    """观察模式：只记录越界流量，全部放行。"""
    def request(self, flow: http.HTTPFlow) -> None:
        host = flow.request.pretty_host
        if _allowed(host):
            return
        with open(LOG, "a") as f:
            f.write(f"{time.strftime('%F %T')} PASS {host} {flow.request.path[:120]}\n")

addons = [Egress()]
```

用 `-s` 挂到 mitmdump 上跑：

```bash
sudo mitmdump -p 8080 -s /opt/egress/egress_proxy.py
```

这里有个坑我先替你踩了：**别在脚本里自己再起一个 `DumpMaster` 然后调 `asyncio.run()`**。`mitmdump -s` 本身已经在跑事件循环，脚本里再起一个会直接抛 `RuntimeError: asyncio.run() cannot be called from a running event loop`，代理连端口都监听不上。写成上面这种「只定义 addon + 末尾 `addons = [...]`」的形式就行。

**先跑观察模式，至少一周。** 这很关键——直接上拦截会打爆所有人的工位，而且你根本不知道有什么东西在偷偷依赖某个域名。

一周后，把日志聚合成一份「实际出网清单」：

```bash
# 按域名统计访问次数，从高到低
awk '{print $4}' /var/log/ai-egress.log | sort | uniq -c | sort -rn | head -40
```

拿这份清单和业务方过一遍：哪些是构建必需的，加进 `ALLOW`；哪些是某个工具在偷偷传东西，不用商量，直接转拦截。

然后把观察模式改成拦截模式，只加一个 `flow.response`：

```python
    def request(self, flow: http.HTTPFlow) -> None:
        host = flow.request.pretty_host
        if _allowed(host):
            return
        with open(LOG, "a") as f:
            f.write(f"{time.strftime('%F %T')} BLOCK {host} {flow.request.path[:120]}\n")
        flow.response = http.Response.make(
            403,
            b"blocked by egress policy\n",
            {"Content-Type": "text/plain"},
        )
```

部署的时候有几点要注意：

- 让客户端强制走代理。只发环境变量不可靠，用系统级代理配置、PAC 文件，或者在网关上做透明代理
- 处理证书固定的客户端。这类工具会直接报错而不是降级，需要单独列进例外并配审计，别指望它乖乖让你解密
- 白名单要按域名，不要按 IP。云服务的 IP 段是动态的，按 IP 维护会把人累死
- 保留日志。出事的时候，「谁在什么时间把什么送出去了」这份日志就是唯一的证据。承明科技能发那封函，靠的就是独立取证

## 最后

回头看整件事，我觉得最值得记住的不是智谱这一家。

而是这个顺序：一个开发者在清理磁盘时发现文件夹不对劲 → 拆开客户端 → 还原出完整上传链路 → 全网才知道。

没有监管，没有安全审计，没有第三方评估。

而在那之前，那套加密上传机制已经跑了不知道多久——每个登录的账号都会触发，两个「看起来相关」的开关关掉也拦不住，删掉的包半小时后自动重生成。

对开发者来说，这件事能带走的东西其实很简单：**把对工具的信任，从「相信」改成「可验证」。**

你不需要成为安全研究员。你只要会三件事——看进程有没有出网、看请求体有多大、看本地有没有多出打不开的大文件。

这三件事花不了十分钟，但它们是你目前唯一能自己掌控的那部分。

---

*参考资料：ferstar 对 ZCode 客户端的逆向分析（9/18）、智谱 ZCode 官方致歉声明（9/18）、太原承明科技公开发函及红星资本局采访（9/19-20）、华尔街见闻与新浪财经关于 Agent 数据行为的行业梳理（9/19）、cereblab 对 Grok Build 的抓包分析、Anthropic 关于 Claude Code 的公开确认。文中 macOS 命令（`pgrep` / `lsof` / `nettop` / `top -pid` / `du` / `find` / `chflags`）已在 Mac mini（M4）上实测；两个 mitmproxy 插件在 mitmproxy 12.2.3 上加载运行通过。*
