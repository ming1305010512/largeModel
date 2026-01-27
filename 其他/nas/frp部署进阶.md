## 1）CentOS 安装 Nginx

### CentOS 7/8 通用（先装 EPEL）

```
sudo yum install -y epel-release
sudo yum install -y nginx httpd-tools
```

启动并设置开机自启：

```
sudo systemctl enable nginx
sudo systemctl start nginx
```

检查：

```
sudo systemctl status nginx
```

------

## 2）（强烈建议）给 frp 加 token

### ECS 的 `frps.ini`

编辑你的 `frps.ini`：

```
[common]
bind_port = 7000
token = 这里填一个很长很随机的token
```

重启 frps（你现在是手动运行的话就 Ctrl+C 再重新运行）：

```
./frps -c frps.ini
```

### 你电脑的 `frpc.ini`

```
[common]
server_addr = 47.113.191.162
server_port = 7000
token = 同一个token

[web]
type = tcp
local_ip = 127.0.0.1
local_port = 8080
remote_port = 6006
```

（如果你文件管理是另一个端口，比如 5244，再加一个）

```
[filemgr]
type = tcp
local_ip = 127.0.0.1
local_port = 5244
remote_port = 6007
```

------

## 3）让 Nginx 把 80 转发到 frp 的远端端口

在 CentOS 上新建 Nginx 配置文件：

```
sudo vi /etc/nginx/conf.d/frp.conf
```

粘贴这份（先不搞域名，直接用 IP 跑通）：

```
server {
    listen 80;
    server_name _;

    # 你的 Web（外网访问 http://ECS_IP/）
    location / {
        proxy_pass http://127.0.0.1:6006;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # 你的文件管理（外网访问 http://ECS_IP/files/）
    # 如果你暂时不需要文件管理，就先把这一段注释掉
    location /files/ {
        auth_basic "Restricted";
        auth_basic_user_file /etc/nginx/.htpasswd;

        proxy_pass http://127.0.0.1:6007/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

------

## 4）给 /files/ 加个访问密码（强烈建议）

```
sudo htpasswd -c /etc/nginx/.htpasswd yourname
```

它会让你输入两次密码。

### 在 Nginx 里怎么用

```
location / {
    auth_basic "Restricted Area";
    auth_basic_user_file /etc/nginx/.htpasswd;
}
```

然后：

```
nginx -t
systemctl reload nginx
```

------

## 5）检测并重载 Nginx

```
sudo nginx -t
sudo systemctl reload nginx
```

## 6）安全组 / 防火墙放行建议（非常关键）

你现在至少要在 ECS 安全组放行：

- `7000`（frp 控制通道，本地 frpc 连接用）
- `80`（给浏览器访问）
- （可选）`6006/6007` 以后建议别暴露，**只留 Nginx 80 即可**

✅ 最稳做法：

- **开放：80、7000**
- **关闭：6006、6007**（让它们只在 ECS 本机 127.0.0.1 用）

> 你截图里现在开放了 6006/7000/8080，后面我们可以把 6006/8080 关掉，只留 80 和 7000。

如果 CentOS 自己还有 firewalld（有的镜像默认开）：

```
sudo systemctl status firewalld
```

如果开着，放行 80/7000：

```
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --permanent --add-port=7000/tcp
sudo firewall-cmd --reload
```

------

## 7）怎么验证成功？

1. 你电脑本地先确认服务正常：
   - `http://127.0.0.1:8080` 你自己的 Web 能打开
2. frps 在 ECS 正常运行
3. frpc 在你电脑显示 `start proxy success`
4. 浏览器访问：
   - `http://47.113.191.162/`  应该就是你本地 8080 的 Web
   - `http://47.113.191.162/files/` 会弹密码（如果你配置了 filemgr）