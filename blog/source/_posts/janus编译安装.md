---
title: janus安装
---

```
使用centos环境
```



## 下载安装

**安装依赖**

```bash
yum install -y epel-release && \
yum update -y && \
yum install -y deltarpm && \
yum install -y openssh-server which file curl zip unzip wget && \
yum install -y libmicrohttpd-devel jansson-devel libnice-devel glib22-devel opus-devel libogg-devel pkgconfig  gengetopt libtool autoconf automake make gcc gcc-c++ git cmake libconfig-devel openssl-devel
yum install -y autoconf automake libtool
```

**下载编译libsrtp 1.5.4**

```bash
wget https://github.com/cisco/libsrtp/archive/v1.5.4.tar.gz
tar xfv v1.5.4.tar.gz
cd libsrtp-1.5.4
./configure --prefix=/usr --enable-openssl --libdir=/usr/lib64
make shared_library && make install

cd -

wget https://github.com/cisco/libsrtp/archive/v2.0.0.tar.gz
tar xfv v2.0.0.tar.gz
cd libsrtp-2.0.0
./configure --prefix=/usr --enable-openssl
make shared_library && make install
```

**下载编译sofia-sip for sip-gateway plugin**

```bash
cd -
wget https://sourceforge.net/projects/sofia-sip/files/sofia-sip/1.12.11/sofia-sip-1.12.11.tar.gz
tar zxf sofia-sip-1.12.11.tar.gz && cd sofia-sip-1.12.11 && ./configure --prefix=/usr/local CFLAGS=-fno-aggressive-loop-optimizations && make && make install
```

**说明**

```
PKG_CONFIG_PATH=/usr/local/lib/pkgconfig/ 这个路径需要与后面janus编译时候设置的参数保持一直
```



**下载编译usrsctp for Data channel support**

```bash
cd -
git clone https://github.com/sctplab/usrsctp && cd usrsctp && \
./bootstrap && \
./configure --prefix=/usr && make && make install
```

**下载编译libwebsocket for android or ios instead of http/https**

```
cd -
git clone https://github.com/warmcat/libwebsockets && \
mkdir libwebsockets/build && cd libwebsockets/build && \
cmake -DMAKE_INSTALL_PREFIX:PATH=/usr -DCMAKE_C_FLAGS="-fpic" .. && \
make && make install
```

**下载安装janus**

```bash
cd -
git clone https://github.com/meetecho/janus-gateway.git && \
cd janus-gateway &&\
sh autogen.sh && \
./configure --prefix=/opt/janus --disable-rabbitmq --disable-docs --disable-libsrtp2 PKG_CONFIG_PATH=/usr/local/lib/pkgconfig/  &&\
make && make install && make configs
```

**说明**

```
从github上拉相关包的时候太慢了，可以通过码云同步到码云后下载zip包然后在编译安装。
PKG_CONFIG_PATH=/usr/local/lib/pkgconfig/ 这个路径需要与sofia-sip的安装路径保持一致
```

**无法找到libwebsockets错误解决**

```
ln -s /usr/local/lib/*.so /usr/lib
ln -s /usr/local/lib/*.so.14 /usr/lib
ldconfig
```

