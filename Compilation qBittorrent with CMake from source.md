# Compilation qBittorrent with CMake from source

This how-to will guide you though the compilation of qBittorrent and `` libtorrent-rasterbar ``.
This guide is written for CentOS, but the process should be similar for other RHEL distributions.

## Required dependencies

### General required dependencies

```bash
yum install epel-release -y
yum install autoconf automake gcc gcc-c++ git glib2 glibc gmp gnutls libblkid libcap libffi libgcc libgcrypt libgpg-error libicu libidn2 libmount libselinux libstdc++ libtasn1 libtool libunistring libuuid lz4-libs make nettle openssl-devel openssl-libs p11-kit pcre pcre2 qt5-qtbase systemd-libs tar wget xz-libs zlib -y
yum install qt5-linguist qt5-qttools-devel qt5-qtsvg-devel -y
yum install screen -y
```

To prevent all kinds of accidents with the server, use screen to create a session

```bash
screen -S qBittorrent
```

### Boost

Download latest version of Boost Version ``1.78.0 December 8th, 2021 03:45 GMT``

```bash
wget https://boostorg.jfrog.io/artifactory/main/release/1.78.0/source/boost_1_78_0.tar.gz
```

Compile:

```bash
# If you are using a vps with only one core, then ignore -j$(( $(nproc) - 1 )), the same below
export DIR_BOOST="/opt/boost"
tar -xvf boost_1_78_0.tar.gz
cd boost_1_78_0
./bootstrap.sh --prefix=${DIR_BOOST}
./b2 install --prefix=${DIR_BOOST} --with=all -j$(( $(nproc) - 1 ))
```

Notice: ``-j$(( $(nproc) - 1 ))`` is a option to use multi-threaded compilation via ``$(nproc)``, it will report an error if your computer has only one core.

### Libtorrent

`` Libtorrent `` is a library written by Arvid Norberg that qBittorrent depends on. It is necessary to compile and install libtorrent before compiling qBittorrent.

Clone from the repository:  ``git clone --depth 1 -b RC_1_2 https://github.com/arvidn/libtorrent.git``

Compile:

```bash
cd libtorrent
./autotool.sh
./configure --prefix=/usr --disable-debug --enable-encryption --with-boost=${DIR_BOOST}
make -j$(( $(nproc) - 1 ))
make install
ln -s /usr/lib/pkgconfig/libtorrent-rasterbar.pc /usr/lib64/pkgconfig/libtorrent-rasterbar.pc
```

Last command was missing and on 64bit systems will fail without it. Here is the error information:

```
checking for libtorrent... no
configure: error: Package requirements (libtorrent-rasterbar >= 1.0.6) were not met:

No package 'libtorrent-rasterbar' found

Consider adjusting the PKG_CONFIG_PATH environment variable if you
installed software in a non-standard prefix.

Alternatively, you may set the environment variables libtorrent_CFLAGS
and libtorrent_LIBS to avoid the need to call pkg-config.
See the pkg-config man page for more details.
```

## Compiling qBittorrent (without the GUI)

First, obtain the qBittorrent source code.

Either download and extract a .tar archive from the GitHub releases page or clone the git repository: ``git clone --depth 1 -b v4_3_x https://github.com/qbittorrent/qBittorrent``

Compile:

```bash
cd qBittorrent
./configure --prefix=/usr --disable-gui CPPFLAGS=-I/usr/include/qt5 --with-boost=${DIR_BOOST}
make -j$(( $(nproc) - 1 ))
make install
```

Since you disabled the graphical user interface, qBittorrent can only be controlled via its WebUI. By default, you can access it at ``http://localhost:8080`` with the default credentials:

```
Username: admin
Password: adminadmin
```

## Control qBittorrent via systemctl

In order to control qBittorrent using systemctl, a .service file needs to be written
```bash
vi /etc/systemd/system/qbittorrent.service
```

```
[Unit]
Description=qbittorrent torrent server

[Service]
User=root
ExecStart=/usr/bin/qbittorrent-nox
Restart=on-abort

[Install]
WantedBy=multi-user.target
```

## Add ProxyPass in Apache

It is convenient to use port 8080 directly. In order to use ssl certificate and better security, a reverse proxy is used here.

```
ProxyPass "/qbt/" "http://127.0.0.1:8080/"
ProxyPassReverse "/qbt/" "http://127.0.0.1:8080/"
```

## Troubleshooting

If are you facing a problem like this:

```
qbittorrent-nox: error while loading shared libraries: libtorrent-rasterbar.so 10: cannot open shared object file: No such file or directory
```

This often happened when you are using 64-bit CentOS 7.x. And it's because of the libraries that the qBittorrent need are not in ``/usr/lib64/``.

You can simply create a soft link to solve it. Do it like this:

```bash
ln -s /usr/lib/libtorrent-rasterbar.so.10 /usr/lib64/libtorrent-rasterbar.so.10
```

For missing ``libboost_system.so.1.78.0``:

```bash
ln -s /opt/boost/lib/libboost_system.so.1.78.0 /usr/lib64/libboost_system.so.1.78.0
```
