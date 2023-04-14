# Complete Process For Setuping a V2Fly Server with TLS + WebSocket

- [Complete Process For Setuping a V2Fly Server with TLS + WebSocket](#complete-process-for-setuping-a-v2fly-server-with-tls--websocket)
  - [1. Something need to be prepared](#1-something-need-to-be-prepared)
    - [1.1. Some choice of server provider](#11-some-choice-of-server-provider)
    - [1.2. Some choice of domain provider](#12-some-choice-of-domain-provider)
  - [2. Preparing for server setup](#2-preparing-for-server-setup)
    - [2.1. Switch the yum repo from CentOS 8 to CentOS 8 Steam](#21-switch-the-yum-repo-from-centos-8-to-centos-8-steam)
    - [2.2. Enable yum-plugin-fastestmirror plugin](#22-enable-yum-plugin-fastestmirror-plugin)
    - [2.3. Update the System and install some necessary software](#23-update-the-system-and-install-some-necessary-software)
    - [2.4. Enable Google BBR(Reboot is required to take effect)](#24-enable-google-bbrreboot-is-required-to-take-effect)
    - [2.5. (Optional) Install cockpit for easy server management](#25-optional-install-cockpit-for-easy-server-management)
  - [3.Install V2Fly via official script](#3install-v2fly-via-official-script)
  - [4. Install and configure apache2](#4-install-and-configure-apache2)
    - [4.1 Install apache2](#41-install-apache2)
    - [4.2 Using acme.sh to install SSL cert](#42-using-acmesh-to-install-ssl-cert)
    - [4.3 Install cert](#43-install-cert)
    - [4.4. Configure certs in Apache](#44-configure-certs-in-apache)
    - [4.5. Denies all http access and redirects to elsewhere](#45-denies-all-http-access-and-redirects-to-elsewhere)
    - [4.6. Configure Reverse Proxy to V2Ray in httpd](#46-configure-reverse-proxy-to-v2ray-in-httpd)
    - [4.7. Configure the V2Ray Server](#47-configure-the-v2ray-server)
  - [5. Client's choice](#5-clients-choice)
  - [6. Afterword](#6-afterword)
  - [7. Citation](#7-citation)

## 1. Something need to be prepared

- a server with "freedom" Internet connction
- a domain
- Some basic operation and maintenance technologies
- ability to debug
- Read more documents if you need~

If you one of lack the above conditions, I will list some options below

### 1.1. Some choice of server provider

Since the Internet service providers and network conditions in each region are different, here are only a few recommended servers that have been used:

- [Vultr](https://www.vultr.com/)
- [DigitalOccon](https://www.digitalocean.com/)
- [Linode](https://www.linode.com/)
- [Bandwagonhos](https://bandwagonhost.com/)
- [Greebclould](https://greencloudvps.com/)
- [V.PS](https://vps.hosting/)
- [AWS](https://aws.amazon.com/)
- [OCI](https://www.oracle.com/cloud/)
- [Hetzner](https://www.hetzner.com/)

### 1.2. Some choice of domain provider

Considering the usage of domain, Chinese mainland domain provider are not recommended. I recommend NameCheap, although not cheap at all. Of course, in addition to NameCheap, there are options such as NameSilo, GoDaddy, etc.

## 2. Preparing for server setup

Here I use CentOS 9 Stream as the operating system of the server. Some server providers' linux distribution mirrors are not updated and are only provided up to CentOS 8.

### 2.1. Switch the yum repo from CentOS 8 to CentOS 8 Steam

CentOS Linux 8 will reach End Of Life (EOL) on December 31st, 2021.  The CentOS Linux 8 packages have been removed from the mirrors. So you need to switch the yum repo to CentOS Steam

```Shell
# Solutions from the CentOS community
rpm -iv --replacepkgs https://vault.centos.org/8.0.1905/BaseOS/x86_64/os/Packages/centos-release-8.0-0.1905.0.9.el8.x86_64.rpm

sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*

dnf distro-sync
dnf --disablerepo '*' --enablerepo extras swap centos-linux-repos centos-stream-repos
```

### 2.2. Enable yum-plugin-fastestmirror plugin

Add the following to ``/etc/dnf/dnf.conf``

```
fastestmirror=1
max_parallel_downloads=8
```

### 2.3. Update the System and install some necessary software

```shell
yum update -y
yum install epel-release -y
yum install vim nano htop git wget unzip -y
```

### 2.4. Enable Google BBR(Reboot is required to take effect)

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

### 2.5. (Optional) Install cockpit for easy server management

Install and enable cockpit

```shell
yum install cockpit -y
systemctl enable --now cockpit.socket
```

Open the firewall if necessary

```shell
sudo firewall-cmd --permanent --zone=public --add-service=cockpit
sudo firewall-cmd --reload
```

## 3.Install V2Fly via official script

> Bash script for installing V2Ray in operating systems such as Debian / CentOS / Fedora / openSUSE that support systemd

Since most of the server configurations I choose are relatively "energy-saving", the use of docker images is not considered here

```shell
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
systemctl enable --now v2ray
```

## 4. Install and configure apache2

### 4.1 Install apache2

```shell
yum install httpd mod_ssl openssl -y
systemctl enable httpd
```

### 4.2 Using acme.sh to install SSL cert

In this tutorial we install cert in default location. Firstly, make directories and install acme.sh

```shell
yum install tar socat -y
curl https://get.acme.sh | sh
```

Then we can use acme.sh to issue and renew certs automatically.

### 4.3 Install cert

The default certificate issuer of the acme.sh script, ZeroSSL, is not easy to use, and needs to be switched to Let's Encrypt.

```shell
~/.acme.sh/acme.sh --register-account -m your-email-adress@example.com
~/.acme.sh/acme.sh --set-default-ca --server letsencrypt
```

Open port 80/443 for Apache

```shell
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --permanent --add-port=80/udp
firewall-cmd --permanent --add-port=443/udp
firewall-cmd --reload
```

Run the script to get cert file

```shell
mkdir -p /etc/pki/httpd/private
~/.acme.sh/acme.sh --issue -d your-domain.example.com --standalone -k ec-256
~/.acme.sh/acme.sh --renew -d your-domain.example.com --force --ecc
~/.acme.sh/acme.sh --installcert -d your-domain.example.com --fullchainpath /etc/pki/httpd/server.crt --keypath /etc/pki/httpd/private/server.key --ecc
```

### 4.4. Configure certs in Apache

Edit ``/etc/httpd/conf.d/ssl.conf``, change these following lines

```
SSLCertificateFile /etc/pki/httpd/server.crt
SSLCertificateKeyFile /etc/pki/httpd/private/server.key

SSLProtocol             all -SSLv3 -TLSv1 -TLSv1.1
SSLCipherSuite          ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
SSLHonorCipherOrder     off
SSLSessionTickets       off
```

### 4.5. Denies all http access and redirects to elsewhere

Edit ``/etc/httpd/conf.d/ssl.conf``, add the following lines at the end

```
<VirtualHost *:80>
    <IfModule alias_module>
        Redirect permanent / "https://www.youtube.com/watch?v=dQw4w9WgXcQ"
    </IfModule>
</VirtualHost>
```

### 4.6. Configure Reverse Proxy to V2Ray in httpd

Edit ``/etc/httpd/conf.d/ssl.conf``, add the following in ``<VirtualHost _default_:443>``

```
ProxyPass "/ray/" "ws://127.0.0.1:10000/ray/"
ProxyAddHeaders Off
ProxyPreserveHost On
RequestHeader append X-Forwarded-For %{REMOTE_ADDR}s
```

The following is an example from the official tutorial, but I don't want to do this.

```
<Location "/ray/">
  ProxyPass ws://127.0.0.1:10000/ray/ upgrade=WebSocket
  ProxyAddHeaders Off
  ProxyPreserveHost On
  RequestHeader append X-Forwarded-For %{REMOTE_ADDR}s
</Location>
```

### 4.7. Configure the V2Ray Server

Although v2fly has released the v5 standard, the json format of the v5 standard is not enabled by default, so the past v4 format is used here.

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

## 5. Client's choice

For shadowsocks and V2Ray there are many clients that can be used. For PC, I recommend QV2ray, although the project has stopped maintenance, but this is currently the best client with GUI. For iOS devices, shadowrocket purchased from the US store is the best option.

- Windows
  - [Qv2ray](https://github.com/Qv2ray/Qv2ray)
  - [Clash](https://github.com/Dreamacro/clash)
  - [V2rayN](https://github.com/2dust/v2rayN)
- Linux
  - [Qv2ray](https://github.com/Qv2ray/Qv2ray)
  - [Clash](https://github.com/Dreamacro/clash)
- iOS
  - [Shadowrocket](https://apps.apple.com/us/app/shadowrocket/id932747118)

Simple of a V2ray Subscribe link format

```
vmess://eyJhZGQiOiJzaW1wbGUuY29tIiwiYWlkIjowLCJob3N0IjoiIiwiaWQiOiIwMTdmZTZhNy1kM2E5LTQyMGQtOTljNS1lMDQzMGIxZmJiNzEiLCJuZXQiOiJ3cyIsInBhdGgiOiIvcmF5LyIsInBvcnQiOjQ0MywicHMiOiJ2bWVzc0BzaW1wbGUuY29tOjQ0MyIsInNjeSI6ImF1dG8iLCJzbmkiOiIiLCJ0bHMiOiJ0bHMiLCJ0eXBlIjoibm9uZSIsInYiOjJ9
```

Simple of a Clash yaml file format

```
(WIP)
```

## 6. Afterword

As GFWs and the tools to fight them continue to evolve, the "method" is not static. But now "vmess+ws+tls" is the safest. Of course, there are more radical "methods" such as trojan, vless, xtls, but in my opinion these "methods" still need to be continuously improved. At the same time, I observed that there are still a large number of users using ShadowSocks, and the old "method" is not unusable, but has become unstable.

## 7. Citation

1. [collection](https://www.youtube.com/watch?v=dQw4w9WgXcQ)
2. (WIP)