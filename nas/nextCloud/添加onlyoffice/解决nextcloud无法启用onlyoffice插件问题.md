### 🔥 用 `occ` 命令 **先写配置，再启用插件**

> UI 不给你入口，那我们直接从“后门”把配置塞进去。

执行命令进入容器

```
docker exec -u 33 -it nextcloud bash
```

执行

```
cd /var/www/html
```

### **关键一步：手动写 ONLYOFFICE 的 Document Server 地址**

> ⚠️ 这一步就是“破局钥匙”

如果你的 Document Server 是通过 **宿主机 IP + 8082** 访问的：

```
php occ config:system:set onlyoffice DocumentServerUrl --value="http://宿主机IP:8082/"
```

⚠️ 注意：

- 一定要 **http**
- 一定要 **最后有 `/`**
- 用你浏览器能访问到的那个地址

例如：

```
php occ config:system:set onlyoffice DocumentServerUrl --value="http://127.0.0.1:8082/"
```

其实执行该命令，改的文件就是下面这个文件

```
/var/www/html/config/config.php
```

![image-20260127172102618](C:\Users\16532\AppData\Roaming\Typora\typora-user-images\image-20260127172102618.png)

最后：

### 清缓存（不然 UI 还以为你没配置）

```
php occ maintenance:repair
php occ maintenance:mode --off
```