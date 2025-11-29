## 🚀 【实战总结】国内服务器部署透明代理（TUN模式）：从入门到“炸心态”再到完美解决

这篇教程基于我们刚刚经历的所有“磨难”和最终成功的经验总结而成。它不是一篇普通的教程，而是一份**实战避坑指南**。

你可以直接复制下面的内容发布到你的 Blog。

### 前言：为什么要在服务器上搞这个？

你是否遇到过这些痛点：

* **Docker 拉取镜像失败：** `docker pull` 卡死，换了镜像源也没用。
* **AI 开发受阻：** 在服务器用 VS Code / Cursor 远程开发，Copilot 无法连接，Python 调用 OpenAI API 超时。
* **环境配置麻烦：** 给每个软件（Git, Wget, Apt）单独配代理太累，且很多软件不支持 SOCKS5。

> **终极解决方案：** 在服务器上部署 Mihomo (Clash.Meta) 并开启 **TUN 模式（透明代理）**。它能接管服务器网卡的所有流量，让所有软件（包括 Docker 和系统更新）自动“**翻墙**”，无需额外配置。

### 第一步：环境准备

1.  一台国内 Linux 服务器（Ubuntu/Debian/CentOS）。
2.  一个好用的**机场订阅**（需要获取到具体的节点信息）。
3.  一颗耐心（这很重要）。

### 第二步：安装 Docker（国内特供版）

官方脚本在国内通常连不上，直接用阿里云镜像安装：

```bash
# 1. 下载安装脚本
curl -fsSL https://get.docker.com -o install-docker.sh
# 2. 使用阿里云镜像运行
sudo sh install-docker.sh --mirror Aliyun
# 3. 启动并设置自启
sudo systemctl enable --now docker
```

### 第三步：部署 Mihomo (Clash.Meta)

#### 1. 创建目录

```bash
mkdir -p ~/mihomo
cd ~/mihomo
```

#### 2. 编写 `docker-compose.yml`

直接复制以下内容。注意 `network_mode: "host"` 和 `privileged: true` 是开启 TUN 模式必须的。

```yaml
version: '3'
services:
  mihomo:
    image: metacubex/mihomo:latest
    container_name: mihomo
    restart: always
    network_mode: "host"
    volumes:
      - ./config.yaml:/root/.config/mihomo/config.yaml
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    privileged: true
```

#### 3. 编写 `config.yaml` (核心配置)

> **⚠️ 注意：** 这是最容易出错的地方！建议先用纯文本编辑器去掉节点名字里的 Emoji，防止乱码。**必须**把 SSH 端口放行，否则启动就失联。

```yaml
mixed-port: 7890
allow-lan: true
bind-address: '*'
mode: rule
log-level: info
ipv6: true
tun:
  enable: true
  stack: mixed             # 强烈建议使用 mixed，兼容性比 system 好
  auto-route: true
  auto-detect-interface: true
  dns-hijack:
    - any:53
external-controller: 0.0.0.0:9090
secret: "123456"           # 设置你的管理面板密码
external-ui: ui
external-ui-url: "https://github.com/MetaCubeX/metacubexd/archive/refs/heads/gh-pages.zip"
dns:
  enable: true
  listen: 0.0.0.0:1053
  enhanced-mode: fake-ip   # 推荐 fake-ip，最不容易出死循环
  fake-ip-range: 198.18.0.1/16
  default-nameserver:
    - 223.5.5.5
  nameserver:
    - https://dns.alidns.com/dns-query
    - https://doh.pub/dns-query
proxies:
  # 在这里填入你的节点配置 (Trojan/Vless/Hysteria2 等)
  # ...
proxy-groups:
  - name: Proxies
    type: select
    proxies:
      - 你的节点名字
      - DIRECT
rules:
  - DST-PORT,22,DIRECT     # 【救命规则】必须放在第一行！防止SSH断连
  - GEOSITE,cn,DIRECT
  - GEOIP,CN,DIRECT
  - MATCH,Proxies
```

### 踩坑记录 & 解决方案（本文精华）

我们在部署过程中遇到了无数个坑，以下是血泪总结：

#### 🕳️ 坑一：Docker 镜像拉不下来

  * **现象：** 配置好了 Docker，但 `docker compose up` 时一直卡在 Pulling，最后报错 `context deadline exceeded`。国内 Docker 镜像源目前极其不稳定。
  * **解决：**
    1.  利用外部 SOCKS5 代理（比如本地电脑或其他机器）给 Docker 守护进程注入代理。
    2.  创建配置目录：`sudo mkdir -p /etc/systemd/system/docker.service.d`
    3.  写入代理：
        ```bash
        # 把 IP:PORT 换成你可用的代理
        sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf <<EOF
        [Service]
        Environment="HTTP_PROXY=socks5://ip:port"
        Environment="HTTPS_PROXY=socks5://ip:port"
        EOF
        ```
    4.  重启 Docker：`sudo systemctl daemon-reload && sudo systemctl restart docker`
    > **⚠️ 注意：** 镜像拉取成功后，务必删除这个文件并重启 Docker，否则后续 Mihomo 运行会出问题。

#### 🕳️ 坑二：配置文件格式错误（无限重启）

  * **现象：** `docker ps` 显示容器状态一直是 `Restarting`。
  * **原因：** 直接在终端粘贴配置文件时，把 `cat > file <<EOF` 这种命令头也粘贴进去了；配置文件里包含 Emoji（国旗），在某些 SSH 客户端下乱码，导致解析失败。
  * **解决：** 使用 `head -n 5 config.yaml` 检查文件头部；尽量使用不带 Emoji 的纯净配置。

#### 🕳️ 坑三：Ping 得通，Curl 不通（地狱级难题）

  * **现象：** 容器启动了，`ping google.com` 能通（显示 `198.18.x.x`，延迟极低）；但 `curl -v https://www.google.com` 卡在 `Trying...` 不动，最后超时。
  * **原因：** 这是 Linux 内核的 RP\_Filter (反向路径过滤) 或协议栈冲突导致的。内核认为 TUN 网卡回来的包“路径不对”，直接丢弃了 TCP 握手包。
  * **解决（三板斧）：**
    1.  开启 IP 转发：
        ```bash
        echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
        sudo sysctl -p
        ```
    2.  切换 Mihomo 协议栈（最有效）：在 `config.yaml` 中，将 `stack: system` 改为 `stack: mixed`。如果 `mixed` 还不行，改为 `stack: gvisor`（Google 的用户态栈，兼容性最强）。
    3.  重启容器：`docker compose restart`。

#### 🕳️ 坑四：Web 面板打不开

  * **现象：** 容器绿了，但浏览器访问 `http://ip:9090/ui` 超时。
  * **原因：** 云服务器的安全组（防火墙）没开端口。
  * **解决：** 去云厂商控制台，在安全组入站规则里放行 TCP 9090。

### 最终验证

验证直连：`curl -I https://www.baidu.com` -> 秒回 `200 OK`。
验证代理：`curl -I https
面板管理：访问 `http://ip:9090/ui`，测速并选择绿色节点。

至此，你的服务器已经拥有了“**魔法**”，Docker 拉取、AI 编程、系统更新将畅通无阻！

```