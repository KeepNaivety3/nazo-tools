# 如何充分利用你的VPS - 在完成本职工作的同时还能继续压榨

## 1. 准备工作

### 1.1. 餐前小点

所需要的技能
  - 一个可以连接到互联网的服务器
  - 一个域名
  - 一些基础 Linux 操作技能及 debug 能力
  - 提问的智慧

一些可供选择的服务器提供商
  - 真正的大厂：可以无脑选择，就是偏贵
    - [AWS](https://aws.amazon.com/)
    - [GCP](https://cloud.google.com/)
    - [Azure](https://azure.microsoft.com/)
    - [OCI](https://www.oracle.com/cloud/)
  - 小有名气的供应商：也可以选择，价格相较大厂便宜点
    - [Vultr](https://www.vultr.com/)
    - [DigitalOccon](https://www.digitalocean.com/)
    - [Linode](https://www.linode.com/)
    - [Greebclould](https://greencloudvps.com/)
    - [Hetzner](https://www.hetzner.com/)
    - [OVH](https://www.ovhcloud.com/)
    - [Bandwagonhost](https://bandwagonhost.com/)

### 1.2. 拿到服务器后需要做的一点点小小的准备

这里我使用 CentOS Stream 9 作为服务器的操作系统，其他 Linux 发行版也能用，但是我就喜欢 RH 系的。

#### 1.2.1. 如果你的发行版过老，请切换到新的发行版吧

如果你的服务器供应商没有最新的 CentOS Stream 9，也可以使用 CentOS Stream 8。但是对于 CentOS 8 用户，坑爹的红帽已经停止更新了，需要切换到 CentOS Stream 8。

> CentOS Linux 8 will reach End Of Life (EOL) on December 31st, 2021. The CentOS Linux 8 packages have been removed from the mirrors.

```bash
# 来自 CentOS 社区的解决方案
rpm -iv --replacepkgs https://vault.centos.org/8.0.1905/BaseOS/x86_64/os/Packages/centos-release-8.0-0.1905.0.9.el8.x86_64.rpm

sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*

dnf distro-sync
dnf --disablerepo '*' --enablerepo extras swap centos-linux-repos centos-stream-repos
```

#### 1.2.2. 开启 yum-plugin-fastestmirror 插件

为了节约几秒钟时间，请在 `/etc/dnf/dnf.conf` 中添加以下内容

```
fastestmirror=1
max_parallel_downloads=8
```

#### 1.2.3. 更新软件包并安装必要的软件包

```bash
yum update -y
yum install epel-release -y
yum install vim nano htop git wget unzip bash-completion net-tools tree -y
```

#### 1.2.4. 开启 Google BBR

> BBR ("Bottleneck Bandwidth and Round-trip propagation time") is a new congestion control algorithm developed at Google. Congestion control algorithms — running inside every computer, phone or tablet connected to a network — that decide how fast to send data.

BBR 听说挺好用的，反正开了没坏处

```bash
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p
```

检验你的 BBR 是否真的开了

```
sysctl -n net.ipv4.tcp_congestion_control
lsmod | grep bbr
```

#### 1.2.5 系统调优 TuneD

安装软件

```bash
yum install tuned -y
```

查看推荐的配置文件

```bash
tuned-adm recommend
```

应用指定配置文件，这里以我认为最好的 `throughput-performance` 为例

```bash
tuned-adm profile throughput-performance
```

## 2. 安装 Nginx & podman