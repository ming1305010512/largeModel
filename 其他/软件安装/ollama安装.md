## 一、下载

https://github.com/ollama/ollama/releases

![image-20260120011845011](C:\Users\16532\AppData\Roaming\Typora\typora-user-images\image-20260120011845011.png)

## 二、安装zstd

```
apt-get update
apt-get install -y zstd
```

## 三、解压

```
tar --use-compress-program=unzstd -xvf ollama-linux-amd64.tar.zst
```

## 四、安装

```
sudo install -m 755 /tmp/bin/ollama /usr/local/bin/ollama
```

## 五、启动

```
OLLAMA_HOST=0.0.0.0:11434 ollama serve
```

## 六、测试

```
curl http://127.0.0.1:11434/api/tags
```

