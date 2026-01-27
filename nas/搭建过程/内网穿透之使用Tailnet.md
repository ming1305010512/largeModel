[TOC]

好 👍
我们一步一步来，我按**最稳、最少坑、可回退**的方式带你装。

下面假设你现在是在 **NAS 本机的 Ubuntu Server 22.04** 上，用的是 `lm` 用户（有 sudo）。

------

## 一、在 NAS 上安装 Tailscale（官方方式）

### ① 一条命令安装（推荐）

在 NAS 终端直接执行：

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

你会看到类似：

```
Adding tailscale repository...
Installing tailscale...
```

安装完成后，检查一下：

```bash
tailscale version
```

能看到版本号就说明 **安装成功** ✅

------

## 二、启动 Tailscale 并登录

### ② 启动服务

```bash
sudo tailscale up
```

这一步会发生两件事：

- 终端会提示你一个 **登录链接**
- 同时会在后台启动 tailscaled 服务

你会看到类似：

```
To authenticate, visit:

https://login.tailscale.com/a/xxxxxxxxxxxx
```

------

### ③ 在你现在用的电脑浏览器里打开这个链接

> ⚠️ 注意：
> **不是在 NAS 上打开**，
> 而是在你现在这台能上网的电脑 / 手机上打开。

登录你的 Tailscale 账号（Google / GitHub / Microsoft 都行）。

登录成功后，你会看到：

- 新设备加入
- 设备名类似：`nas`, `ubuntu`, `lm-nas` 等

------

## 三、确认 NAS 已成功接入 Tailnet

回到 NAS 终端，执行：

```bash
tailscale status
```

如果看到类似：

```
100.xxx.xxx.xxx   nas   lm@   linux   active
```

👉 **恭喜，你的 NAS 已经在 Tailscale 私网里了**

------

## 四、从你的电脑 SSH 连 NAS（关键一步）

### ④ 在你现在这台电脑上

先装 Tailscale（如果你还没装）
👉 https://tailscale.com/download

登录同一个账号。

然后在电脑终端执行（示例）：

```bash
ssh lm@100.xxx.xxx.xxx
```

或者用设备名（如果系统支持）：

```bash
ssh lm@nas
```

如果你能直接 SSH 进去：

🎉 **内网穿透这一步你已经 100% 搞定了**

------

## 五、强烈推荐你立刻做的 2 个小加固（很重要）

### ✅ 1️⃣ 开机自动启动（一般已默认开启，但确认一下）

```bash
systemctl status tailscaled
```

如果是 `active (running)` 就不用管了。

------

### ✅ 2️⃣ 只允许 Tailscale IP 访问 SSH（可选但推荐）

等你确认 **Tailscale 访问完全 OK** 之后，可以这样做：

```bash
sudo nano /etc/ssh/sshd_config
```

确认 / 修改以下内容：

```conf
PasswordAuthentication yes
ListenAddress 0.0.0.0
```

👉 后续我们可以升级为：

- 只监听 `tailscale0`
- 或关闭密码登录，改用 key

**但现在先不动，稳住优先**

------

## 六、回答你刚才隐含的那个“高手问题”

> **“我有 VPS，还装了 Tailscale，会不会冲突？”**

结论一句话：

> ❌ 不冲突
> ✅ 但现在没必要用 VPS

Tailscale：

- 是“管理面 + 私网”
- 非侵入式
- 你以后 **一定会保留**

FRP：

- 是“公网暴露”
- 用的时候再上

------

## 你现在已经完成了什么（很关键）

到这一步，你已经拥有：

- ✅ 一个**永远能远程登录的 NAS**
- ✅ 不怕换网络 / 没公网 IP
- ✅ 不需要路由器折腾
- ✅ 不暴露任何端口

**这是 NAS 的“生命线”

