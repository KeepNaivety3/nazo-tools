# Complete Process For Setuping a V2Fly Server with TLS + WebSocket

## Some ohoice of server provider

Since the Internet service providers and network conditions in each region are different, here are only a few recommended servers that have been used:

- Sugerhosts <https://www.sugarhosts.com/>
	- Hong Kong server, lower latency for most regions
	- 1 vCPU
	- 1GB RAM
	- 20GB storage
	- 20Mbit/s Port (IPv6 is not supported)
	- 300GB bandwidth
	- CN¥99 MONTHLY, Support Alipay payment

- DigitalVM <https://digital-vm.com/>
	- Storage VM and Power VM with japan node
	- 1 vCPU / 2 vCPU
	- 512MB RAM / 1GB RAM
	- 30GB storage / 20GB storage
	- 1Gbit/s Port / 10Gbit/s Port
	- 5TB bandwidth / 20TB bandwidth
	- \$8 MONTHLY / \$13 MONTHLY,  Support Alipay payment

- Greebclould <https://greencloudvps.com/>
	- Japan SSD KVM VPS
	- 1 vCPU
	- 1GB RAM
	- 15GB storage
	- 1Gbit/s Port
	- 1TB bandwidth
	- \$6 MONTHLY, Support Alipay payment

- Vultr <https://www.vultr.com/>
	- \$6 MONTHLY
	- The configuration is similar, but it will be blocked by GFW in "special periods"

- 搬瓦工 <https://bandwagonhost.com/>
	- I haven't used it, but the CN2 GIA has good reviews
	- Only support annual payment

## Domain name purchase

Considering the reasons for the need for filing, domestic domain name buyers are not recommended. I recommend namecheap, although not cheap at all.

## Preparing for server setup

### Switch the yum repo to CentOS Steam

CentOS Linux 8 will reach End Of Life (EOL) on December 31st, 2021.  The CentOS Linux 8 packages have been removed from the mirrors. So you need to switch the yum repo to CentOS Steam

```Shell
# Solutions from the CentOS community
rpm -iv --replacepkgs https://vault.centos.org/8.0.1905/BaseOS/x86_64/os/Packages/centos-release-8.0-0.1905.0.9.el8.x86_64.rpm

sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*

dnf distro-sync
dnf --disablerepo '*' --enablerepo extras swap centos-linux-repos centos-stream-repos
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

### Enable Google BBR(Reboot is required to take effect)

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

### (Optional) Install cockpit for easy server management

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

## Install V2Fly via official script

> Bash script for installing V2Ray in operating systems such as Debian / CentOS / Fedora / openSUSE that support systemd

Since most of the server configurations I choose are relatively "energy-saving", the use of docker images is not considered here

```shell
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
systemctl enable --now v2ray
```

## Install and configure apache2

### Install apache2

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

### Configure certs in Apache

Edit ``/etc/httpd/conf.d/ssl.conf``
Change these following lines

```
SSLCertificateFile /etc/pki/httpd/server.crt
SSLCertificateKeyFile /etc/pki/httpd/private/server.key
```

### Redirect all http to https

Edit ``/etc/httpd/conf.d/ssl.conf``

Add the following lines at the end

```
<VirtualHost *:80>
    <IfModule alias_module>
        Redirect permanent / https://example.com/
    </IfModule>
</VirtualHost>
```

### Configure Reverse Proxy to V2Ray in httpd

Edit ``/etc/httpd/conf.d/ssl.conf``
Add the following in ``<VirtualHost _default_:443>``

```
ProxyPass "/ray/" "ws://127.0.0.1:10000/ray/"
ProxyAddHeaders Off
ProxyPreserveHost On
RequestHeader append X-Forwarded-For %{REMOTE_ADDR}s
```

### Configure the V2Ray Server

Although v2fly has released the v5 standard, the json format of the v5 standard is not enabled by default, so the past v4 format is used here。

Edit ``/usr/local/etc/v2ray/config.json``

```
{
  "inbounds": [
    {
      "protocol": "vmess",
      "settings": {
        "clients": [
          {
            "id": "********-****-****-****-************"
          }
        ]
      },
      "port": "10000",
      "listen": "127.0.0.1",
      "tag": "",
      "sniffing": {},
      "streamSettings": {
        "transport": "ws",
        "transportSettings": {
          "path": "/ray/",
        },
        "security": "none",
        "securitySettings": {}
      }
    }
  ],
  "outbounds": [
    {
        "protocol": "freedom",
        "settings": {},
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

## Client's choice(WIP)

For shadowsocks and V2Ray there are many clients that can be used. For PC, I recommend QV2Ray, although the project has stopped maintenance, but this is currently the best client with GUI. For iOS devices, shadowrocket purchased from the US store is the best option.

### Windows

- Qv2ray https://github.com/Qv2ray/Qv2ray
- clash https://github.com/Dreamacro/clash
- V2rayN https://github.com/2dust/v2rayN

### Linux

- Qv2ray https://github.com/Qv2ray/Qv2ray
- clash https://github.com/Dreamacro/clash

### iOS

- Shadowrocket https://apps.apple.com/us/app/shadowrocket/id932747118

## Afterword

WIP