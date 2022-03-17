# 搭建一个WebSocket + TLS + Web的V2Fly服务器

[TOC]

## 服务器的选择

由于每个地区的网络服务商与网络状况都由区别，这里只推荐几个使用过的比较好用的服务器：

- 糖果云 <https://www.sugarhosts.com/>
	- 香港地区服务器 低延迟 
	- 1 vCPU
	- 1GB RAM
	- 20GB storage
	- 20Mbit/s Port (IPv6 is not supported)
	- 300GB bandwidth
	- CN¥99 MONTHLY, Support Alipay payment

- DigitalVM <https://digital-vm.com/>
	- 东京服务器 Storage VM和Power VM均可,  黑五等节日有折扣
	- 1 vCPU / 2 vCPU
	- 512MB RAM / 1GB RAM
	- 30GB storage / 20GB storage
	- 1Gbit/s Port / 10Gbit/s Port
	- 5TB bandwidth / 20TB bandwidth
	- \$8 MONTHLY / \$13 MONTHLY,  Support Alipay payment

- Greebclould <https://greencloudvps.com/>
	- Japan SSD KVM VPS,  黑五等节日有折扣
	- 1 vCPU
	- 1GB RAM
	- 15GB storage
	- 1Gbit/s Port
	- 1TB bandwidth
	- \$6 MONTHLY, Support Alipay payment

- Vultr <https://www.vultr.com/>
	- \$6 MONTHLY
	- 配置类似，但是重大日子容易被墙

- DigitalOcean <https://www.digitalocean.com/>
	- 没用过，但是评价不错

- 搬瓦工 <https://bandwagonhost.com/>
	- 没用过，但是CN2 GIA的评价不错
	- 只支持年付

## 域名的购买

Considering the reasons for the need for filing, domestic domain name buyers are not recommended. I recommend namecheap, although not cheap at all.

## Preparing for server setup

### Switch the yum repo to CentOS Steam

CentOS Linux 8 will reach End Of Life (EOL) on December 31st, 2021.  The CentOS Linux 8 packages have been removed from the mirrors. So you need to switch the yum repo to CentOS Steam

```Shell
# Solutions from the CentOS community
rpm -iv --replacepkgs https://vault.centos.org/8.0.1905/BaseOS/x86_64/os/Packages/centos-release-8.0-0.1905.0.9.el8.x86_64.rpm

sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*

dnf --disablerepo '*' --enablerepo extras swap centos-linux-repos centos-stream-repos
dnf distro-sync
```

### Enable yum-plugin-fastestmirror plugin

Add the following to ``/etc/dnf/dnf.conf``

```
fastestmirror=1
max_parallel_downloads=8
```

### Update the System and install some necessary software

```shell
yum update -y
yum install epel-release -y
yum install vim nano htop git -y
```

### Enable Google BBR

BBR ("Bottleneck Bandwidth and Round-trip propagation time") is a new congestion control algorithm developed at Google. Congestion control algorithms — running inside every computer, phone or tablet connected to a network — that decide how fast to send data.

```shell
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p
```

Verify that BBR is enabled successfully
```shell
sysctl -n net.ipv4.tcp_congestion_control
lsmod | grep bbr
```

### Install cockpit for easy server management

Install cockpit

```shell
yum install cockpit -y
```

Enable cockpit

```shell
systemctl enable --now cockpit.socket
```

Open the firewall if necessary

```shell
sudo firewall-cmd --permanent --zone=public --add-service=cockpit
sudo firewall-cmd --reload
```

After completing the preparations, reboot the system for the next installation

## (Optional) Install Shadowsocks-libev

Shadowsocks is an "Internet tool" of the previous generation. When the 80/443 port of the server is blocked, it can be used temporarily. You can get a lot of one-click installation scripts by using search engines, but for the sake of safety, here is a way to make and install Shadowsocks-libev. Better shadowsocks-rust has been released, since shadowsocks is used very rarely, I did not study the installation method of the latest rust version.

Install some necessary software

```shell
yum install gcc gettext autoconf libtool automake make pcre-devel asciidoc xmlto c-ares-devel libev-devel libsodium-devel mbedtls-devel -y
```

Download the source code of shadowsocks-libev

```shell
git clone https://github.com/shadowsocks/shadowsocks-libev.git
cd shadowsocks-libev
git submodule update --init --recursive
```

Start compiling

```
./autogen.sh && ./configure --prefix=/usr && make
make install
```
Configure shadowsocks-libev

```shell
mkdir -p /etc/shadowsocks-libev
vim /etc/shadowsocks-libev/config.json
```
```
{
	"server":["::0","0.0.0.0"],
	"server_port":*PORT*,
	"local_port":1080,
	"password":"*PASSWORD*",
	"timeout":60,
	"method":"aes-256-gcm"
}
```

Set up to start automatically

```shell
vim /etc/systemd/system/shadowsocks.service
```

```
[Unit]
Description=Shadowsocks Server
After=network.target

[Service]
ExecStart=/usr/bin/ss-server -c /etc/shadowsocks-libev/config.json -u
Restart=on-abort

[Install]
WantedBy=multi-user.target
```
```shell
systemctl enable shadowsocks
```
Open firewall ports

```shell
firewall-cmd --permanent --add-port=*PORT*/*(tcp/udp)*
firewall-cmd --reload
```

Shadowsocks is already available, and you can view the service status at the same time

```shell
systemctl start shadowsocks
systemctl status shadowsocks
```

## Install V2Fly via official script

> Bash script for installing V2Ray in operating systems such as Debian / CentOS / Fedora / openSUSE that support systemd

It is not recommended to use this project to install v2ray in docker, please use the official image directly. <https://github.com/v2fly/docker>

```shell
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
systemctl enable --now v2ray
```

## Install and configure apache

### Install apache

```shell
yum install httpd mod_ssl openssl -y
systemctl enable httpd
```

### Using acme.sh to install SSL cert

In this tutorial we install cert in default location. Firstly, make directories and install acme.sh

```shell
yum install tar socat -y
curl https://get.acme.sh | sh
```
Then we can use acme.sh to issue and renew certs automatically.

### Install cert

~~Using ZeroSSL.com CA need register. ZeroSSL doesn't have rate limits. One can issue unlimited TLS/SSL certificate valid for 90 days (ref).~~

```shell
~/.acme.sh/acme.sh --register-account  -m myemail@example.com
~/.acme.sh/acme.sh --set-default-ca --server letsencrypt
```
Open port 80/443
```shell
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --permanent --add-port=80/udp
firewall-cmd --permanent --add-port=443/udp
firewall-cmd --reload
```

```shell
mkdir -p /etc/pki/httpd/private
~/.acme.sh/acme.sh --issue -d example.com --standalone -k ec-256
~/.acme.sh/acme.sh --renew -d example.com --force --ecc
~/.acme.sh/acme.sh --installcert -d example.com --fullchainpath /etc/pki/httpd/server.crt --keypath /etc/pki/httpd/private/server.key --ecc
```

### Configure Certs in Apache

Edit ``/etc/httpd/conf.d/ssl.conf``
Change these following lines

```
SSLCertificateFile /etc/pki/httpd/server.crt
SSLCertificateKeyFile /etc/pki/httpd/private/server.key
```

### Redirect all http to https

Edit ``/etc/httpd/conf.d/ssl.conf``

```
<VirtualHost *:80>
    <IfModule alias_module>
        Redirect permanent / https://keepnaive233.network/
    </IfModule>
</VirtualHost>
```

### Configure Reverse Proxy to V2Ray in httpd

Edit ``/etc/httpd/conf.d/ssl.conf``
Add the following in ``<VirtualHost>``

```
<Location "/ray/">
	ProxyPass "/ray/" "ws://127.0.0.1:10000/ray/"
	ProxyAddHeaders Off
	ProxyPreserveHost On
	RequestHeader append X-Forwarded-For %{REMOTE_ADDR}s
</Location>
```

### Configure the V2Ray Server

Edit ``/usr/local/etc/v2ray/config.json``

```
{
  "inbounds": [
    {
      "port": 10000,
      "listen":"127.0.0.1",
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

For privacy, I hided the UUID. You may generate yours by [Online UUID Generator](https://www.uuidgenerator.net/).
After configuring, restart V2Ray and httpd.

```shell
/usr/sbin/setsebool -P httpd_can_network_connect 1
systemctl restart v2ray httpd
```

## Client's choice

For shadowsocks and V2Ray there are many clients that can be used. For PC, I recommend QV2Ray, although the project has stopped maintenance, but this is currently the best client with GUI. For iOS devices, shadowrocket purchased from the US store is the best option.

### Windows

- Qv2ray https://github.com/Qv2ray/Qv2ray
- clash https://github.com/Dreamacro/clash
- V2rayN https://github.com/2dust/v2rayN

### Linux

- Qv2ray https://github.com/Qv2ray/Qv2ray
- clash https://github.com/Dreamacro/clash
- v2rayA https://github.com/v2rayA/v2rayA

### Android

- SagerNet https://github.com/SagerNet/SagerNet
- v2rayNG https://github.com/2dust/v2rayNG

### iOS

- Shadowrocket https://apps.apple.com/us/app/shadowrocket/id932747118

## Afterword

随着GFW与“上网工具”的对抗，“上网工具”也有版本的迭代。从最开始的ShadowSock到后来的V2Ray VMess，目前更高级的工具包括V2Ray VLESS XTLS和trojan。目前使用的V2Ray即将更新v5版本。“上网工具”的及时更新与迭代时很有必要的。GFW可以看成一个黑盒，具有某些特征的流量将被阻碍，但目前并不清楚它的“工具”识别原理，现在的做法是将上网流量用tls伪装成对自己域名的访问。
