### 前言：为什么要在服务器上搞这个？

你是否遇到过这些技术痛点：

* **Docker 镜像拉取困难：** `docker pull` 过程卡顿或失败，更换国内镜像源效果不佳。
* **海外技术服务调用受阻：** 在服务器用远程开发环境（如 VS Code / Cursor）时，智能代码辅助功能（如 Copilot）连接不稳定；Python 调用某些海外技术服务 API 超时。
* **环境配置冗余：** 需为每个软件（如 Git, Wget, Apt）单独配置代理，工作量大，且部分工具不支持标准的 SOCKS5 协议。

> **终极解决方案：** 在服务器上部署 Mihomo (Meta) 核心并开启 **TUN 模式（透明流量接管）**。它能接管服务器网卡的所有流量，让所有软件（包括 Docker 和系统更新）**自动使用统一的网络配置**，无需对每个应用进行额外设置。

---

### 第一步：环境准备

1.  一台国内 Linux 服务器（Ubuntu/Debian/CentOS）。
2.  一份符合要求的网络配置订阅（需获取到具体的节点信息，以兼容 Mihomo 核心）。
3.  一颗耐心（这很重要）。

### 第二步：安装 Docker（国内特供版）

官方脚本在国内部分网络环境下可能连接受阻，建议直接使用国内镜像安装：

```bash
# 1. 下载安装脚本
curl -fsSL [https://get.docker.com](https://get.docker.com) -o install-docker.sh

# 2. 使用阿里云镜像运行
sudo sh install-docker.sh --mirror Aliyun

# 3. 启动并设置自启
sudo systemctl enable --now docker
````

### 第三步：部署 Mihomo (Meta 核心)

#### 1\. 创建目录

```bash
mkdir -p ~/mihomo
cd ~/mihomo
```

#### 2\. 编写 `docker-compose.yml`

直接复制以下内容。注意 `network_mode: "host"` 和 `privileged: true` 是开启 TUN 模式接管流量必须的配置。

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

#### 3\. 编写 `config.yaml` (核心配置)

> **⚠️ 注意：** 这是最容易出错的地方！在粘贴节点配置前，建议先用纯文本编辑器处理节点名称中的特殊字符（如 Emoji），防止乱码导致解析失败。**必须**将服务器 SSH 端口（默认为 22）设置为直连规则，否则服务启动后可能导致 SSH 断连。

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
external-ui: uiexternal-ui-url: [https://github.com/MetaCubeX/metacubexd/archive/refs/heads/gh-pages.zip](https://github.com/MetaCubeX/metacubexd/archive/refs/heads/gh-pages.zip)
dns:
  enable: true
  listen: 0.0.0.0:1053
  enhanced-mode: fake-ip   # 推荐 fake-ip，最不容易出死循环
  fake-ip-range: 198.18.0.1/16
  default-nameserver:
    - 223.5.5.5
  nameserver:
    - [https://dns.alidns.com/dns-query](https://dns.alidns.com/dns-query)
    - [https://doh.pub/dns-query](https://doh.pub/dns-query)
proxies:
  # 在这里填入你的节点配置 (如 V2/Shadow/Hyste 等)
  # ...
proxy-groups:
  - name: GlobalSelect
    type: select
    proxies:
      - 你的节点名字
      - DIRECT
rules:
  - DST-PORT,22,DIRECT     # 【救命规则】必须放在第一行！防止SSH断连
  - GEOSITE,cn,DIRECT
  - GEOIP,CN,DIRECT
  - MATCH,GlobalSelect
```

-----

### 踩坑记录 & 解决方案（本文精华）

我们在部署过程中遇到了无数个坑，以下是血泪总结：

### ☠️ 坑零：启动即失联（最致命）

**现象**：

  * 刚运行 `docker compose up -d`，SSH 终端立刻卡死，服务器直接失联。
  * 重启服务器后能连上，但只要一启动容器就又断了。

**原因**：

  * 开启 TUN 模式（透明代理）后，Mihomo 会接管网卡的所有流量，**包括你当前连接服务器的 SSH 流量（端口 22）**。
  * 如果 Mihomo 把 SSH 流量也尝试转发给代理节点，或者在路由过程中处理不当，你的连接通道会被瞬间切断。

**解决（救命稻草）**：
在 `config.yaml` 的 `rules` 部分，**第一行**必须强制放行 SSH 端口：

```yaml
rules:
  # ⚠️ 必须放在第一行！告诉 Mihomo：SSH 流量别动，直接放行！
  - DST-PORT,22,DIRECT
  
  # ... 其他规则 ...
```

**保命技巧**：
在调试阶段，建议养成使用“后悔药”命令的习惯。在启动容器前执行 `sudo shutdown -r +10`（10分钟后自动重启）。万一失联了，等 10 分钟服务器重启，你就能重新连进去删掉容器修配置了。

#### 🕳️ 坑一：Docker 镜像拉不下来

  * **现象：** 配置好了 Docker，但 `docker compose up` 时一直卡在 Pulling，最后报错 `context deadline exceeded`。国内 Docker 镜像源在拉取海外镜像时目前极其不稳定。
  * **解决：**
    1.  利用外部 SOCKS5 代理（比如本地电脑或其他机器）给 Docker 守护进程注入代理。
    2.  创建配置目录：`sudo mkdir -p /etc/systemd/system/docker.service.d`
    3.  写入代理：
        ```bash
        # 把 IP:PORT 换成你可用的 SOCKS5 代理
        sudo tee /etc/systemd/system/docker.service.d/http-proxy.conf <<EOF
        [Service]
        Environment="HTTP_PROXY=socks5://ip:port"
        Environment="HTTPS_PROXY=socks5://ip:port"
        EOF
        ```
    4.  重启 Docker：`sudo systemctl daemon-reload && sudo systemctl restart docker`
    > **⚠️ 注意：** 镜像拉取成功后，务必删除这个文件并重启 Docker，否则后续 Mihomo 运行时会因为网络配置冲突而出现问题。

#### 🕳️ 坑二：配置文件格式错误（无限重启）

  * **现象：** `docker ps` 显示容器状态一直是 `Restarting`。
  * **原因：** 直接在终端粘贴配置文件时，可能粘贴了非 YAML 内容；配置文件里包含特殊字符（如 Emoji），在某些 SSH 客户端下乱码，导致 YAML 解析失败。
  * **解决：** 使用 `head -n 5 config.yaml` 检查文件头部；尽量使用不带特殊字符的纯净配置。

#### 🕳️ 坑三：Ping 得通，Curl 不通（地狱级难题）

  * **现象：** 容器启动了，`ping google.com` 能通；但 `curl -v https://www.google.com` 卡在 `Trying...` 不动，最后超时。
  * **原因：** 这是 Linux 内核的 RP\_Filter (反向路径过滤) 或协议栈冲突导致的，内核丢弃了 TUN 网卡返回的 TCP 握手包。
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
  * **原因：** 云服务器的安全组（防火墙）没有开放端口。
  * **解决：** 去云厂商控制台，在安全组入站规则里放行 TCP 9090。

-----

### 最终验证

| 验证项 | 命令 | 预期结果 |
| :--- | :--- | :--- |
| **验证直连** | `curl -I https://www.baidu.com` | 秒回 `200 OK` |
| **验证配置** | `curl -I https://www.google.com` | 秒回 `200 OK` |
| **面板管理** | 访问 `http://ip:9090/ui` | 成功打开面板，测速并选择可用的网络节点。 |

至此，你的服务器已经拥有了**统一的网络流量管理能力**，Docker 拉取、海外技术服务调用、系统更新将畅通无阻！

```
```
