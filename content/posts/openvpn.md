---
weight: 2
title: "利用 openVPN 构建互访混合云环境"
date: 2021-01-17T19:18:26+08:00
draft: false
categories: ["技术"]
tags: ["openVPN"]
---

## 背景

由于业务需要，公司内部一个 License 服务器需要与部分部署在公有云私有子网的服务进行互访 check in。

<!--more-->

但后来经过调研发现了一个问题，就是我们的公有云服务商是某个外国云的中国区环境，并不提供原生 VPN 隧道服务，且由于时间比较赶来不及申请专线资源。

经过讨论，最终决定选择利用 openVPN 将本地局域网以及公有云上的私有子网打通，作为临时方案。

&nbsp;

## 拓扑

![拓扑](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/openvpn/openvpn.png)

本地子网：172.16.11.0/24

公有云 VPC 子网：10.20.0.0/16

隧道网段：10.8.0.0/24

&nbsp;

## 部署 Server

### 准备工作

修改`/etc/selinux/config`配置文件，关闭 selinux

```bash
SELINUX=disabled
```

关闭防火墙

```bash
systemctl disable firewalld
systemctl stop firewalld
```

### 安装 openvpn

```bash
yum -y install epel-release
yum update
yum -y install openvpn easy-rsa
```

### 创建证书

```bash
cd /usr/share/easy-rsa/3/
./easyrsa init-pki
./easyrsa build-ca nopass
./easyrsa build-server-full <server-name> nopass
./easyrsa build-client-full <client-name> nopass
```

将`<server-name>`和`<client-name>`分别替换为服务端和客户端的证书名，`nopass`则表示不使用密码加密证书，client 可以创建多个，如果多个设备同时使用一个客户端证书，向服务器申请时会获取到同一个 IP 地址，最终造成互相冲突，所以通常是每个设备创建一个客户端证书。

### 配置 server

将证书复制到 openvpn目录`/etc/openvpn/server/`，然后创建`dh.pem`和`ta.key`

```bash
cd /usr/share/easy-rsa/3/pki
cp ca.crt issued/<server-name>.crt private/<server-name>.key /etc/openvpn/server/
cd /etc/openvpn/server
openssl dhparam -out dh.pem 2048
openvpn --genkey --secret ta.key
```

在`/etc/openvpn/server/`中创建一个配置文件`<server-name>.conf`

```bash
port 1994
proto udp                                    
dev tun
ca /etc/openvpn/server/ca.crt
cert /etc/openvpn/server/server.crt
key /etc/openvpn/server/server.key
dh /etc/openvpn/server/dh.pem
server 10.8.0.0 255.255.255.0                #隧道IP
push "route 10.20.0.0 255.255.0.0"           #公有云VPC子网网段
client-config-dir /etc/openvpn/ccd            #手动创建，而且建立客户端名称文件
route 192.168.3.0 255.255.255.0               #客户端网络，与ccd下客户端文件对应
route 172.16.11.0 255.255.255.0              #客户端网络，与ccd下客户端文件对应
ifconfig-pool-persist ipp.txt
keepalive 10 120
max-clients 100
status openvpn-status.log
verb 3
client-to-client                            #允许客户端之间互访
log /var/log/openvpn.log
persist-key
persist-tun
```

在服务器端`/etc/openvpn/`下创建 CCD 目录，添加客户端配置文件

```bash
cat /etc/openvpn/ccd/clientvm
iroute 192.168.3.0 255.255.255.0
iroute 172.16.11.0 255.255.255.0
```

配置 IP 转发

```bash
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
systemctl restart network
```

配置 NAT，为了避免重启后失效，建议写到`rc.local`里面

```bash
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -j MASQUERADE
iptables -L -n -t nat
```

### 启动 openvpn server

可以使用`systemctl enable`设置为开机自启动

```bash
systemctl start openvpn-server@<server-name>
systemctl enable openvpn-server@<server-name>
```

### 公有云配置

禁用源/目标端检查，否则无法转发

![禁用源端检查](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/openvpn/yuanduanjiancha.png)

可以为需要访问本地服务的子网直接添加路由表，避免每个实例逐条添加

![路由表](https://wumanhoblogimg.obs.cn-south-1.myhuaweicloud.com/images/openvpn/routingtable.png)

&nbsp;

## 部署 Client

客户端的安装部署方式跟 Server 是一样的

1. 关闭 SELINUX
2. 关闭防火墙
3. 安装 openvpn

```bash
yum -y install epel-release
yum update
yum -y install openvpn
```

### 配置客户端

openvpn 客户端和服务端都是同一个程序，只是根据配置文件来区分。

安装好 openvpn 之后，需要将在服务器生成的`ca.crt`、`ta.key`、`<client-name>.key`、`<client-name>.crt`复制到`/etc/openvpn/client/`目录下，然后在该目录下创建一个`<client-name>.conf`配置文件。

```bash
client #指定当前VPN是客户端
dev tun #使用tun隧道传输协议
proto udp #使用udp协议传输数据
remote  52.83.152.63 1994  #openvpn服务器IP地址端口号
resolv-retry infinite #断线自动重新连接，在网络不稳定的情况下非常有用
remote-cert-tls server
auth-nocache 
nobind #不绑定本地特定的端口号
ca /etc/openvpn/client/ca.crt #指定CA证书的文件路径
cert /etc/openvpn/client/clientvm.crt #指定当前客户端的证书文件路径
key /etc/openvpn/client/clientvm.key #指定当前客户端的私钥文件路径
verb 3 #指定日志文件的记录详细级别，可选0-9，等级越高日志内容越详细
persist-key #通过keepalive检测超时后，重新启动VPN，不重新读取keys，保留第一次使用的keys
persist-tun #检测超时后，重新启动VPN，一直保持tun是linkup的。否则网络会先linkdown然后再linkup
status openvpn-status.log    				#日志
log /var/log/openvpn/openvpn.log 			#按需配置
```

配置 IP 转发

```bash
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
systemctl restart network
```

配置 NAT，为了避免重启后失效，建议写到`rc.local`里面

```bash
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -j MASQUERADE
iptables -L -n -t nat
```

### 启动客户端

```bash
systemctl start openvpn-client@<client-name>
systemctl enable openvpn-client@<client-name>
```

### 本地业务服务器配置静态路由

本地 License 等服务器需要通过 openvpn client 转发数据到公有云，所以需要在每个服务器上添加一条静态路由

```bash
route add -net 10.20.0.0/16 gw 172.16.11.16  ### 指向VPN client 的IP为路由
```

&nbsp;

（完）

&nbsp;

### 参考

[OpenVPN client端配置文件详细说明](https://www.xiexianbin.cn/linux/vpn/2016-10-28-openvpn-client-config/index.html)



