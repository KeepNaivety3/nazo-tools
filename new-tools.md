# 如何充分利用你的VPS - 在完成本职工作的同时还能继续压榨

- [如何充分利用你的VPS - 在完成本职工作的同时还能继续压榨](#如何充分利用你的vps---在完成本职工作的同时还能继续压榨)
  - [1. 准备工作](#1-准备工作)
    - [1.1. 拿到服务器后需要做的一点点小小的准备](#11-拿到服务器后需要做的一点点小小的准备)
      - [1.2.1. 从 CentOS 切换到滚动更新的 CentOS Stream (已过时)](#121-从-centos-切换到滚动更新的-centos-stream-已过时)
      - [1.2.2. 开启 yum-plugin-fastestmirror 插件与多线程下载](#122-开启-yum-plugin-fastestmirror-插件与多线程下载)
      - [1.2.3. 更新软件包并安装必要的软件包](#123-更新软件包并安装必要的软件包)
      - [1.2.4. 开启 Google BBR](#124-开启-google-bbr)
      - [1.2.5 系统调优 TuneD](#125-系统调优-tuned)
      - [1.2.6. 使用 chrony 校时](#126-使用-chrony-校时)
      - [1.2.7. 修改服务器 Hostname](#127-修改服务器-hostname)
  - [2. 安装“正事”](#2-安装正事)
    - [2.1. 安装 Nginx \& Podman](#21-安装-nginx--podman)
    - [2.2. 安装 V2Fly](#22-安装-v2fly)
    - [2.3. 安装证书](#23-安装证书)
    - [2.4. 编辑 nginx 配置文件](#24-编辑-nginx-配置文件)
    - [2.5. 额外的几个工作（选作）](#25-额外的几个工作选作)
      - [2.5.1. 为你的 V2Fly 添加 ipv6 支持](#251-为你的-v2fly-添加-ipv6-支持)
      - [2.5.2.  为你的容器设置开机自启](#252--为你的容器设置开机自启)
  - [3. 继续压榨服务器的剩余价值](#3-继续压榨服务器的剩余价值)
    - [3.1. 安装 Cockpit](#31-安装-cockpit)
    - [3.2. 安装 Code-Server](#32-安装-code-server)
    - [3.3. 安装 AList](#33-安装-alist)
    - [3.4. 安装 qBittorrent-nox](#34-安装-qbittorrent-nox)
    - [3.5. 使用 Nginx Autoindex 访问服务器文件](#35-使用-nginx-autoindex-访问服务器文件)
  - [4. 客户端的选用](#4-客户端的选用)

## 1. 准备工作

### 1.1. 拿到服务器后需要做的一点点小小的准备

这里我使用 CentOS Stream 9 作为服务器的操作系统，其他 Linux 发行版也能用。

#### 1.2.1. 从 CentOS 切换到滚动更新的 CentOS Stream (已过时)

如果你的服务器供应商没有最新的 CentOS Stream 9，也可以使用 CentOS Stream 8。但是对于 CentOS 8 用户，红帽已经停止更新了，需要切换到 CentOS Stream 8。

> CentOS Linux 8 will reach End Of Life (EOL) on December 31st, 2021. The CentOS Linux 8 packages have been removed from the mirrors.

```bash
# 来自 CentOS 社区的解决方案
rpm -iv --replacepkgs https://vault.centos.org/8.0.1905/BaseOS/x86_64/os/Packages/centos-release-8.0-0.1905.0.9.el8.x86_64.rpm

sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*

dnf distro-sync
dnf --disablerepo '*' --enablerepo extras swap centos-linux-repos centos-stream-repos
```

#### 1.2.2. 开启 yum-plugin-fastestmirror 插件与多线程下载

对于那些网络条件不太好的vps可以选择开启

> The fastest mirror plugin is designed for use in repository configurations where you have more than 1 mirror in a repo configuration. It makes a connection to each mirror, timing the connection and then sorts the mirrors by fastest to slowest for use by yum.

```
echo "fastestmirror=1" >> /etc/dnf/dnf.conf
echo "max_parallel_downloads=8" >> /etc/dnf/dnf.conf
```

#### 1.2.3. 更新软件包并安装必要的软件包

部分是必需品，另一部分是 debug 用的方便的。

```bash
yum update -y
yum install -y epel-release
yum install -y vim nano htop git wget unzip bash-completion net-tools tree
```

#### 1.2.4. 开启 Google BBR

> BBR ("Bottleneck Bandwidth and Round-trip propagation time") is a new congestion control algorithm developed at Google. Congestion control algorithms — running inside every computer, phone or tablet connected to a network — that decide how fast to send data.

BBR 是谷歌开发的新型 TCP 拥塞控制算法。在此以前，互联网主要使用基于丢包的拥塞控制策略，只依靠丢失数据包的迹象作为减缓发送速率的信号。BBR尝试通过使用全新的拥塞控制来解决这个问题，它使用基于延迟而不是丢包作为决定发送速率的主要因素。对于高丢包率与高时延的网络环境，相较于 Linux 默认的 Cubic 算法具有更高的带宽。
关于 Google BBR 的相关内容跟可以参考[来自谷歌的产品介绍](https://cloud.google.com/blog/products/networking/tcp-bbr-congestion-control-comes-to-gcp-your-internet-just-got-faster)和[这篇来自 AWS 的博客](https://aws.amazon.com/cn/blogs/china/talking-about-network-optimization-from-the-flow-control-algorithm/)。

![谷歌 BBR 算法介绍图例](https://storage.googleapis.com/gweb-cloudblog-publish/original_images/GCP-TCP-BBR-animate-r32B252812529plh0.GIF)

BBR 从 4.9 版本开始就已经出现在 Linux 内核之中，可以通过一个简单的 sysctl 命令来启用。

```bash
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p
```

校验你的 `BBR` 是否开启，如果返回的结果中带有 `bbr` 说明已经生效。

```
sysctl -n net.ipv4.tcp_congestion_control
lsmod | grep bbr
```

#### 1.2.5 系统调优 TuneD

`tuned-adm` 是一个命令行工具，可让您在 Tuned 配置集间切换以提高特定用例的性能。它还提供了 `tuned-adm recommend` 子命令，用于评估您的系统并输出推荐的调优配置文件。具体内容可以参考[红帽给出的文档](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/9/html/monitoring_and_managing_system_status_and_performance/getting-started-with-tuned_monitoring-and-managing-system-status-and-performance)。

```bash
yum install tuned -y
```

查看推荐的配置文件

```bash
tuned-adm recommend
```

应用指定配置文件，针对 vps 可以使用 `virtual-guest`，为 Red Hat Enterprise Linux 9 虚拟机和 VMWare 虚拟机设计的配置集基于 `throughput-performance` 配置集（除其他任务）减少了虚拟内存的交换性并增加磁盘预读值。它不会禁用磁盘障碍。它继承 `throughput-performance` 配置集，该配置集将 `energy_performance_preference` 和 `scaling_governor` 属性更改为 `performance` 配置集。

```bash
tuned-adm profile virtual-guest
```

#### 1.2.6. 使用 chrony 校时

这次我们使用的 `VMess` 依赖于系统时间，请确保使用 `V2Ray` 的系统 UTC 时间误差在 90 秒之内，时区无关。

```bash
yum install chrony -y
systemctl enable --now chronyd
```
修改 `/etc/chrony.conf` 为服务器添加额外的 `ntp` 服务器。

这里提供几个可供参考的 ntp 服务器：

```
# Google Public NTP
server time1.google.com iburst
server time2.google.com iburst
server time3.google.com iburst
server time4.google.com iburst

# NTP Pool Project (以亚洲-中国为例)
server 0.asia.pool.ntp.org
server 1.asia.pool.ntp.org
server 2.asia.pool.ntp.org
server 3.asia.pool.ntp.org

server 0.cn.pool.ntp.org
server 1.cn.pool.ntp.org
server 2.cn.pool.ntp.org
server 3.cn.pool.ntp.org
```

#### 1.2.7. 修改服务器 Hostname

都是你的服务器了，改个主机名吧~

```bash
hostnamectl hostname xxx
```

## 2. 安装“正事”

这里选择是“终极配置”，虽然带宽损耗相较其他方法更高，但目前来看仍然是最稳定的。安装方法采用nginx反向代理+podman容器。为什么这么做呢，因为容器肯定是未来的大势所趋，至于 K8S 或者 K3S 我觉得有点 overkill 了。

### 2.1. 安装 Nginx & Podman

```bash
yum install nginx -y
yum install podman podman-compose -y
systemctl enable nginx
```

### 2.2. 安装 V2Fly

```bash
mkdir /podman && cd /podman
touch container-compose.yml
```

编辑刚刚生成的 `container-compose.yml`

```yaml
version: 1.0
services:
  v2fly:
    image: docker.io/v2fly/v2fly-core:latest
    container_name: V2Fly
    command: run -c /etc/v2fly/config.json
    volumes:
      - /podman/v2ray/config.json:/etc/v2fly/config.json:Z
    ports:
      - "127.0.0.1:10000:10000"
    restart: always
```

写入你的 V2Fly 配置文件
其中 `uuid` 需要你去[生成](https://www.uuidgenerator.net/)

```bash
mkdir /podman/v2ray
touch /podman/v2ray/config.json
```

```json
{
  "inbounds": [
    {
      "port": 10000,
      "listen":"0.0.0.0",
      "protocol": "vmess",
      "settings": {
        "clients": [
          {
            "id": "********-****-****-****-************"
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "wsSettings": {
        "path": "/ray/"
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": {}
    }
  ]
}
```

运行 V2Fly 容器

```bash
cd /podman
podman-compose up -d
```

查看正在运行的容器

```bash
podman ps
```

### 2.3. 安装证书

安装 acme.sh

```bash
yum install tar socat -y
curl https://get.acme.sh | sh
```

acme.sh 脚本默认的证书颁发者 ZeroSSL 不好用，需要切换成 Let's Encrypt

```bash
~/.acme.sh/acme.sh --register-account -m your-email-adress@example.com
~/.acme.sh/acme.sh --set-default-ca --server letsencrypt
```

开放 80/443 端口以便安装证书

```bash
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --reload
```

安装证书

```bash
mkdir -p /etc/pki/nginx/private
~/.acme.sh/acme.sh --issue -d your-domain.example.com --standalone -k ec-256
~/.acme.sh/acme.sh --renew -d your-domain.example.com --force --ecc
~/.acme.sh/acme.sh --installcert -d your-domain.example.com --fullchainpath /etc/pki/nginx/server.crt --keypath /etc/pki/nginx/private/server.key --ecc
```

### 2.4. 编辑 nginx 配置文件

在 `/etc/nginx/nginx.conf` 的 `http {}` 内添加，修改 `your-domain.example.com`

```
server {
  listen 443 ssl;
  listen [::]:443 ssl;
  
  ssl_certificate /etc/pki/nginx/server.crt;
  ssl_certificate_key /etc/pki/nginx/private/server.key;
  ssl_session_timeout 1d;
  ssl_session_cache shared:MozSSL:10m;
  ssl_session_tickets off;
  
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
  ssl_prefer_server_ciphers off;
  
  server_name your-domain.example.com;
  location /ray/ {
    if ($http_upgrade != "websocket") {
      return 404;
    }
    proxy_redirect off;
    proxy_pass http://127.0.0.1:10000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;  
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      }
}
```

重启 nginx，并修改 selinux 权限

```bash
systemctl start nginx
setsebool -P httpd_can_network_connect 1
```

### 2.5. 额外的几个工作（选作）

#### 2.5.1. 为你的 V2Fly 添加 ipv6 支持

在 `podman`/`podman-compose` 中创建的默认网桥并不支持ipv6，需要手动添加

如果你是 `cni` 网桥，则修改 `/etc/cni/` 下的 xxx，否则修改 `/etc/containers/network/` 下的 `podman_default.json`

`cni` 网桥需要修改的部分：
```
"routes": [
    {
      "dst": "0.0.0.0/0"
    },
    {
      "dst": "::/0"
    }
],
"ranges": [
    [
      {
          "subnet": "10.89.1.0/24",
          "gateway": "10.89.1.1"
      }
    ],
    [
      {
          "subnet": "fde0:fee8:4b9:2476::/64",
          "gateway": "fde0:fee8:4b9:2476::1"
      }
    ]
]
```

`containers` 网桥需要修改的部分：
```
"subnets": [
  {
    "subnet": "10.89.0.0/24",
    "gateway": "10.89.0.1"
  },
  {
    "subnet": "fde0:fee8:4b9:2476::/64",
    "gateway": "fde0:fee8:4b9:2476::1"
  }
],
"ipv6_enabled": true,
```

重启容器网桥，或者重启服务器

#### 2.5.2.  为你的容器设置开机自启

具体命令行操作请自行搜索，理论上是

```bash
systemctl enable --now podman.socket
systemctl enable --now podman-auto-update.service
```

但是如果你完成下面 `cockpit` 的安装了就仅需要登陆后点击开机自启动 `Podman`，去服务那里开启 `Podman auto-update service` 即可

## 3. 继续压榨服务器的剩余价值

### 3.1. 安装 Cockpit

> Cockpit is a web-based graphical interface for servers, intended for everyone, especially those who are:
>   - new to Linux (including Windows admins)
>   - familiar with Linux and want an easy, graphical way to administer servers
>   - expert admins who mainly use other tools but want an overview on individual systems
> Thanks to Cockpit intentionally using system APIs and commands, a whole team of admins can manage a system in the way they prefer, including the command line and utilities right alongside Cockpit.

> Cockpit 项目是一个基于 Web 的图形界面，通过使用它，您可以执行系统管理任务，如检查和控制 systemd 服务、管理存储、配置网络、分析网络问题以及检查日志。

反正红帽在 `RHEL9` 中开始推这玩意了，确实可以简化轻度运维难度，安了总没错~

```bash
yum install cockpit cockpit-podman -y
```

修改 `/etc/cockpit/cockpit.conf` 为 Cockpit 配置 nginx 反向代理，修改 `your-domain.example.com`

```
[WebService]
Origins = https://your-domain.example.com wss://your-domain.example.com
ProtocolHeader = X-Forwarded-Proto
UrlRoot=/cockpit
```

修改 nginx 配置文件 `/etc/nginx/nginx.conf`，在 `location /ray/ {}` 下方添加

```
location /cockpit/ {
  proxy_pass https://127.0.0.1:9090/cockpit/;
  proxy_set_header Host $host;
  proxy_set_header X-Forwarded-Proto $scheme;

  proxy_http_version 1.1;
  proxy_buffering off;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
  gzip off;
}
```

```bash
systemctl enable --now cockpit.socket
systemctl restart nginx
```

开启 Cockpit 并重启 nginx 后，访问 `https://your-domain.example.com/cockpit/` 网页控制台

### 3.2. 安装 Code-Server

> Run VS Code on any machine anywhere and access it in the browser.

一个网页版的 VSCode，临时编辑文件总比 vim 好用

在 `/podman/container-compose.yml` 内添加，注意缩进

```yaml
code-server:
  image: docker.io/codercom/code-server:latest
  container_name: Code-Server
  user: 0:0
  environment:
    - USER=coder
  volumes:
    - /podman/code-server/config:/root/.config:Z
    - /podman/code-server/plugin:/root/.local:Z
    - /podman/code-space:/root/project:z
    - /podman/container-compose.yml:/root/project/container-compose.yml:z
  ports:
    - "127.0.0.1:9001:8080"
  restart: always
```

修改 nginx 配置文件 `/etc/nginx/nginx.conf`

```
location /coder/ {
  proxy_pass http://127.0.0.1:9001/;
  proxy_set_header Host $host;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection upgrade;
  proxy_set_header Accept-Encoding gzip;
}
```

```bash
cd /podman && podman-compose up -d
systemctl restart nginx
```

开启 Code-Server 并重启 nginx 后，访问 `https://your-domain.example.com/coder/` web 编辑器
默认密码位于 `/podman/code-server/config/code-server/config.yaml`，修改密码仅需要修改 `config.yaml` 并重启容器即可
其中 `/podman/code-space` 对应容器中的 `/root/project` 是我设置的工作区

### 3.3. 安装 AList

一个支持多种存储的文件列表程序，使用 Gin 和 Solidjs。
功能强大使用十分简单的 web 文件系统，支持 webdav

在 `/podman/container-compose.yml` 内添加，注意缩进

```yaml
alist:
  image: docker.io/xhofe/alist:latest
  container_name: AList
  volumes:
    - /data:/data:z
    - /podman/alist:/opt/alist/data:Z
  ports:
    - "127.0.0.1:5244:5244"
  restart: always
```

修改 nginx 配置文件 `/etc/nginx/nginx.conf`

```
location /alist/ {
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;
  proxy_set_header Host $http_host;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header Range $http_range;
  proxy_set_header If-Range $http_if_range;
  proxy_redirect off;
  proxy_pass http://127.0.0.1:5244/alist/;
  client_max_body_size 100m;
}
```

获取初始登陆密码

```bash
cd /podman && podman-compose up -d
podman logs AList
```

```
Successfully created the admin user and the initial password is: ******** 
```

修改 alist 的配置文件 `/podman/alist/config.json` 并重启容器

```
"site_url": "alist",
```

```bash
podman restart AList
systemctl restart nginx
```

重启 AList 并重启 nginx 后，访问 `https://your-domain.example.com/alist/` web 文件系统
添加了一个本地存储 `/data`，如果安装 qBittorrent 则其默认下载位置也为此，同时 Nginx Autoindex 也会将这个文件夹放出

### 3.4. 安装 qBittorrent-nox

qBittorrent是一个跨平台的开源、自由的BitTorrent客户端，其图形用户界面是通过Qt所写，后端使用libtorrent。

在 `/podman/container-compose.yml` 内添加，注意缩进

```yaml
qbittorrent:
  image: docker.io/qbittorrentofficial/qbittorrent-nox:4.4.5-1
  container_name: qBittorrent-nox
  user: 0:0
  environment:
    - QBT_EULA=true
    - QBT_WEBUI_PORT=8080
  tmpfs:
    - /tmp
  volumes:
    - /podman/qbittorrent:/config:Z
    - /data:/downloads:z
  ports: 
    - "127.0.0.1:8080:8080"
  restart: always
```

修改 nginx 配置文件 `/etc/nginx/nginx.conf`

```
location /qbt/ {
  proxy_pass         http://127.0.0.1:8080/;
  proxy_http_version 1.1;

  proxy_set_header   Host               127.0.0.1:8080;
  proxy_set_header   X-Forwarded-Host   $http_host;
  proxy_set_header   X-Forwarded-For    $remote_addr;

  proxy_cookie_path  /                  "/; Secure";
}
```

```bash
cd /podman && podman-compose up -d
systemctl restart nginx
```

开启 qBittorrent-nox 并重启 nginx 后，访问 `https://your-domain.example.com/qbt/` webui
默认下载位置 `/data` 对应容器中 `/downloads`
容器支持ipv6，因此我认为没有必要再转发多余端口，仅仅多一层NAT

### 3.5. 使用 Nginx Autoindex 访问服务器文件

## 4. 客户端的选用

<Tool
    url="https://github.com/Qv2ray/Qv2ray"
    name="Qv2ray"
    :platforms="['linux','win','mac']"
    description="跨平台 V2Ray 客户端，支持 Linux、Windows、macOS，可通过插件系统支持 SSR / Trojan / Trojan-Go / NaiveProxy 等协议"
/>

<Tool
    url="https://github.com/SagerNet/SagerNet"
    name="SagerNet"
    :platforms="['android']"
    description="SagerNet 是一个基于 V2Ray 的 Android 通用代理应用。"
/>

<Tool
    url="https://github.com/2dust/v2rayN"
    name="V2rayN"
    :platforms="['win']"
    description="V2RayN 是一个基于 V2Ray 内核的 Windows 客户端。"
/>

<Tool
    url="https://github.com/v2rayA/v2rayA"
    name="v2rayA"
    :platforms="['linux','win','mac']"
    description="基于 web GUI 的跨平台 V2Ray 客户端，在 Linux 上支持全局透明代理，其他平台上支持系统代理。"
/>

--- temp ---

```
server {
	listen 443 ssl;
	listen [::]:443 ssl;
	
	ssl_certificate       /etc/pki/nginx/server.crt;
	ssl_certificate_key   /etc/pki/nginx/private/server.key;
	ssl_session_timeout 1d;
	ssl_session_cache shared:MozSSL:10m;
	ssl_session_tickets off;
	
	ssl_protocols         TLSv1.2 TLSv1.3;
	ssl_ciphers           ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
	ssl_prefer_server_ciphers off;
	
	server_name           your-domain.example.com;
	location /ray/ {
		if ($http_upgrade != "websocket") {
			return 404;
		}
		proxy_redirect off;
		proxy_pass http://127.0.0.1:10000;
		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection "upgrade";
		proxy_set_header Host $host;  
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	}
	location /cockpit/ {
		proxy_pass https://127.0.0.1:9090/cockpit/;
		proxy_set_header Host $host;
		proxy_set_header X-Forwarded-Proto $scheme;

		proxy_http_version 1.1;
		proxy_buffering off;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection "upgrade";
		gzip off;
	}
	location /coder/ {
		proxy_pass http://127.0.0.1:9001/;
		proxy_set_header Host $host;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection upgrade;
		proxy_set_header Accept-Encoding gzip;
	}
	location /alist/ {
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto $scheme;
		proxy_set_header Host $http_host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header Range $http_range;
		proxy_set_header If-Range $http_if_range;
		proxy_redirect off;
		proxy_pass http://127.0.0.1:5244/alist/;
		client_max_body_size 100m;
	}
	location /qbt/ {
		 proxy_pass         http://127.0.0.1:8080/;
		 proxy_http_version 1.1;

		 proxy_set_header   Host               127.0.0.1:8080;
		 proxy_set_header   X-Forwarded-Host   $http_host;
		 proxy_set_header   X-Forwarded-For    $remote_addr;

		 proxy_cookie_path  /                  "/; Secure";
	  }
}
```

```
services:
  v2fly:
    image: docker.io/v2fly/v2fly-core:latest
    container_name: V2Fly
    command: run -c /etc/v2fly/config.json
    volumes:
      - /podman/v2ray/config.json:/etc/v2fly/config.json:Z
    ports:
      - "127.0.0.1:10000:10000"
    restart: always
  
  code-server:
    image: docker.io/codercom/code-server:latest
    container_name: Code-Server
    user: 0:0
    environment:
      - USER=coder
    volumes:
      - /podman/code-server/config:/root/.config:Z
      - /podman/code-server/plugin:/root/.local:Z
      - /podman/code-space:/root/project:z
      - /podman/container-compose.yml:/root/project/container-compose.yml:z
    ports:
      - "127.0.0.1:9001:8080"
    restart: always

  alist:
    image: docker.io/xhofe/alist:latest
    container_name: AList
    volumes:
      - /data:/data:z
      - /podman/alist:/opt/alist/data:Z
    ports:
      - "127.0.0.1:5244:5244"
    restart: always
  
  qbittorrent:
    image: docker.io/qbittorrentofficial/qbittorrent-nox:4.4.5-1
    container_name: qBittorrent-nox
    user: 0:0
    environment:
      - QBT_EULA=true
      - QBT_WEBUI_PORT=8080
    tmpfs:
      - /tmp
    volumes:
      - /podman/qbittorrent:/config:Z
      - /data:/downloads:z
    ports: 
      - "127.0.0.1:8080:8080"
    restart: always
```

```
podman rmi $(podman images -a | grep "<none>" | awk '$1=="<none>" {print $3}')
```
