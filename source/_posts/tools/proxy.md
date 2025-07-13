---
title: proxy 
category: tools
---

> 某些国家对网络进行了封锁，需要通过代理来绕过封锁，某些软件可以直接使用操作系统的代理，只需要设置操作系统的代理即可，但某些应用，尤其是命令行工具需要单独设置代理。

代理的设置取决于应用层的协议，比如HTTPS代理，SSH代理等。本地代理软件如v2ray等一般提供HTTP，HTTPS，SOCKS代理协议。下图是ubuntu24的系统代理设置（其中本地代理端口为`127.0.0.1:7897`）。

<img src="https://raw.githubusercontent.com/caohongchuan/blogimg/main/nextimg/image-20250512163458271.png" alt="image-20250512163458271" style="zoom:67%;" />

## 设置命令行代理

```bash
sudo vim ~/.bashrc
```

Add proxy configuration in `~/.bashrc`:

```bash
export HTTP_PROXY="<http://127.0.0.1:7897>"
export HTTPS_PROXY="<http://127.0.0.1:7897>"
```

## GIT 设置代理

在使用GIT时，会用到HTTPS或者SSH协议来下载上传文件，在GIT中两种协议需要分别设置：

全局代理HTTPS：

* 使用http代理：`git config --global http.proxy http://127.0.0.1:58591`

* 使用socks5代理：`git config --global http.proxy socks5://127.0.0.1:51837`

只对Github代理HTTPS：`git config --global http.https://github.com.proxy socks5://127.0.0.1:51837`

全局代理SSH：

```bash
vim ~/.ssh/config
```

```
ProxyCommand nc -v -x 127.0.0.1:7897 %h %p

Host github.com
    User git
    Port 443
    Hostname ssh.github.com
    IdentityFile "/home/amber/.ssh/id_rsa"
    TCPKeepAlive yes

Host ssh.github.com
  User git
  Port 443
  Hostname ssh.github.com
  IdentityFile "/home/amber/.ssh/id_rsa"
  TCPKeepAlive yes
```

其中`ProxyCommand nc -v -x 127.0.0.1:7897 %h %p`中的地址为本地代理地址，`IdentityFile "/home/amber/.ssh/id_rsa"`时SSH密钥地址

## Set npm proxy

> Prerequisite: Set a proxy such as `clash` at [localhost](http://localhost) or remote server.

```bash
npm config set https-proxy <http://id:pass@proxy.example.com>:port
npm config set proxy <http://id:pass@proxy.example.com>:port

# for example 
npm config set https-proxy <http://127.0.0.1:7897>
npm config set proxy <http://127.0.0.1:7897>
```

## Set Electron proxy for download

```bash
sudo vim ~/.bashrc
```

Add `GLOBAL_AGENT_HTTPS_PROXY` to `~/.bashrc`

```bash
# for electron
export ELECTRON_GET_USE_PROXY='true'
export GLOBAL_AGENT_HTTPS_PROXY="<http://127.0.0.1:7897>"
```