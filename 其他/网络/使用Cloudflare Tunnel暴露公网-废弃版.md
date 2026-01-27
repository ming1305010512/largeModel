## 一、 创建域名

以创建好的域名：**leonardchanning.duckdns.org**

步骤：使用DuckDNS创建

打开官网：https://www.duckdns.org，然后创建子域名

## 二、DuckDNS 域名交给 Cloudflare

1. 下载安装Cloudflare

   ```
   winget install -e --id Cloudflare.cloudflared
   ```

   

2. 架构图

   ![cloudflare tunnel 详细介绍—— 面向cloudflare的网站搭建,1分钟 ...](https://img.imoeq.com/i/2023/02/23/image_3.png)

逻辑是：

```
浏览器
  ↓
myapp.duckdns.org   （DNS 在 DuckDNS）
  ↓  CNAME
xxxx.cfargotunnel.com  （Cloudflare Tunnel 提供）
  ↓
Cloudflare Edge
  ↓
Tunnel
  ↓
你本地 localhost:8080
```

3. 正式开始：把 DuckDNS 域名“交给 Cloudflare”

   ### Step 1️⃣：创建 Cloudflare Tunnel（只做一次）

   在你本地终端：

   ```
   cloudflared tunnel login
   ```

   浏览器会打开 Cloudflare，让你授权。

   然后：

   ```
   cloudflared tunnel create my-tunnel
   ```

   你会看到类似输出：

   ```
   Created tunnel my-tunnel
   Tunnel ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx
   ```

   ------

   ### Step 2️⃣：启动一个临时 Tunnel，拿到 Cloudflare 的 hostname

   先不用管 DuckDNS，直接跑：

   ```
   cloudflared tunnel --url http://localhost:8080
   ```

   终端里会出现一行非常关键的地址：

   ```
   https://xxxxxxx.cfargotunnel.com
   ```

   👉 **记住这个 `xxxx.cfargotunnel.com`，下面要用**

   ------

   ### Step 3️⃣：在 DuckDNS 配 CNAME（核心）

   现在回到 **DuckDNS** 页面。

   在你的域名行（`myapp.duckdns.org`）里：

   #### DNS 设置这样填：

   - **IPv4 / IPv6**：留空 或 随便（不重要）

   - **CNAME**：填刚才那条：

     ```
     xxxxxxx.cfargotunnel.com
     ```

   - 点击 **Update IP**

   ✅ 到这里，**DNS 层已经完成**

   ------

   ## 五、验证是否成功（非常关键）

   等 **10～30 秒**，然后在浏览器打开：

   ```
   https://myapp.duckdns.org
   ```

   ### 成功的表现：

   - 页面能打开
   - 显示的是你本地服务
   - HTTPS 自动生效
   - 浏览器证书是 Cloudflare

   🎉 **恭喜，核心目标已经完成**

## 尝试按这套流程接入失败：

原因如下：

好，这句话说得**非常关键**，我们把事情**一次性讲清楚，不再绕**👇

> **你想做的是：把 DuckDNS 创建的域名「接入 Cloudflare」**

我直接给你**标准答案 + 可执行方案 + 为什么你现在会卡**。

------

### 一、先给结论（非常重要）

> **DuckDNS 创建的域名，❌不能作为 Cloudflare 的“站点（Zone）”接入**
> **但 ✅可以“通过 Cloudflare Tunnel 使用 Cloudflare 的网络”**

也就是说：

- ❌ 不能：
  `myname.duckdns.org` → **在 Cloudflare 里 Add a site**
- ✅ 能做：
  `myname.duckdns.org` → **CNAME → Cloudflare Tunnel**

你现在卡住，**不是你不会操作**，而是**这两种“接入”在 Cloudflare 里是两条完全不同的路**。

------

### 二、为什么 DuckDNS 不能“直接接入 Cloudflare”

Cloudflare 所谓的「接入域名」，指的是：

> 👉 **你把域名的 NS（Nameserver）交给 Cloudflare**

而 DuckDNS 的现实是：

- `duckdns.org` 是 **官方的共享域名**

- 你只有一个 **子域名**

  ```
  xxx.duckdns.org
  ```

- **NS 在 duckdns.org 级别**

- 你 **没有权限改 NS**

所以结论是：

> 🚫 **DuckDNS 域名永远不可能成为 Cloudflare Zone**

这也是为什么你在
**Authorize Cloudflare Tunnel** 页面里：

- 看不到任何 zone
- 无法继续
- cert.pem 生成不了

👉 **这是设计限制，不是 bug**

------

### 三、那“接入 Cloudflare”到底还能怎么做？

Cloudflare 官方支持 **3 种层级**（很多教程没说清）：

| 层级               | 是否需要自己域名 | DuckDNS 是否可用 |
| ------------------ | ---------------- | ---------------- |
| Zone（整站接入）   | ✅ 必须           | ❌ 不可           |
| Tunnel（反向代理） | ❌ 不必须         | ✅ 可             |
| Quick Tunnel       | ❌ 不需要         | ✅ 最适合你       |

👉 **你现在只能、也应该走第 2 / 3 种**

------

### 四、你要的正确架构（你现在这一步的“唯一正解”）

### 👉 DuckDNS + Cloudflare Tunnel（CNAME 模式）

逻辑是：

```
浏览器
  ↓
myname.duckdns.org   （DNS 在 DuckDNS）
  ↓  CNAME
xxxxx.cfargotunnel.com  （Cloudflare 提供）
  ↓
Cloudflare 网络
  ↓
Tunnel
  ↓
你本地 localhost:8080
```

这就是：

> **“DuckDNS 域名使用 Cloudflare 网络”**

------

### 五、你现在立刻该做的 3 个步骤（不需要 login）

#### ✅ Step 1：启动 Cloudflare Quick Tunnel

在你本地 CMD / PowerShell：

```bash
cloudflared tunnel --url http://localhost:8080
```

你会看到类似：

```
https://abcd-efgh-ijkl.cfargotunnel.com
```

📌 **这个地址已经是 Cloudflare 提供的公网 HTTPS**



------

#### ✅ Step 2：把 DuckDNS 指向 Cloudflare

回到 DuckDNS 控制台：

- 找到你的域名：`myname.duckdns.org`

- **CNAME** 填：

  ```
  abcd-efgh-ijkl.cfargotunnel.com
  ```

- IPv4 / IPv6：留空或随便

- 点 **Update IP**

------

#### ✅ Step 3：验证

浏览器打开：

```
https://myname.duckdns.org
```

如果看到你本地服务：

🎉 **恭喜，你的 DuckDNS 域名已经“接入 Cloudflare 网络”了**

------

### 六、重要认知校正（帮你避免以后再踩坑）

你现在做到的是：

- ✅ 使用 Cloudflare 的全球边缘网络
- ✅ 使用 Cloudflare 的 HTTPS
- ✅ 隐藏真实 IP
- ❌ 不是 Cloudflare Zone
- ❌ 不能在 Cloudflare 面板管理 DNS / 防火墙规则

如果将来你想要：

- Cloudflare Access
- WAF
- 完整控制
- 正式项目

👉 **那唯一升级方式就是：买一个自己的域名**

