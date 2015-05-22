在CentOS 6下配置L2TP IPsec VPN
==============================

## Prerequisite的安装

```bash
yum install -y ppp iptables make gcc gmp-devel xmlto bison flex xmlto libpcap-devel lsof
```

## Openswan的安装和配置

#### 安装Openswan

```bash
wget http://download.openswan.org/openswan/openswan-2.6.38.tar.gz
tar zxvf openswan-2.6.38.tar.gz
cd openswan-2.6.38
make programs install
```
> 安装 Openswan，记得别用2.6.26，宁可用2.6.24。它和 xL2TPD 存在严重兼容性bug

#### 配置Openswan
编辑**/etc/ipsec.conf** 注意将`$VPS_IP`替换成你机器的IP。这个IP应该是你的公网IP，不是本机内网IP。几处更改如下。
```conf
# which IPsec stack to use. auto will try netkey, then klips then mast
# protostack=auto
protostack=netkey
```

```conf
# 在文件底部添加上如下内容，注意缩进
conn L2TP-PSK-NAT
        rightsubnet=vhost:%priv
        also=L2TP-PSK-noNAT

conn L2TP-PSK-noNAT
        authby=secret
        pfs=no
        auto=add
        keyingtries=3
        rekey=no
        ikelifetime=8h
        keylife=1h
        type=transport
        left=$VPS_IP # 替换IP
        leftid=$VPS_IP # 替换IP
        leftprotoport=17/1701
        right=%any
        rightid=%any
        rightprotoport=17/%any
```

除此之外，需要修改ipsec密钥，连接时需要提供此密钥。创建文件**/etc/ipsec.secrets**，添加内容如下，依然需要替换`$VPS_IP`为你机器的IP，替换`$PASS`为你想指定的密钥。
```conf
$VPS_IP %any: PSK "$PASS"
```

## xL2TPD的安装和配置

#### 安装xL2TPD
在CentOS 6官方的yum源中，没有这个软件包。需要安装fedora的epel源。
```bash
# 32位用户使用此命令: rpm -Uvh http://download.fedoraproject.org/pub/epel/6/i386/epel-release-6-7.noarch.rpm
# 64位用户命令如下
rpm -Uvh http://download.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-7.noarch.rpm
yum install xl2tpd -y
```
> 如果上面这个安装地址如果不对了。参照这个fedora epel 的[faq页](https://fedoraproject.org/wiki/EPEL/FAQ/zh-cn#How_can_I_install_the_packages_from_the_EPEL_software_repository.3F)。

#### 配置xL2TPD
在配置xL2TPD之前，需要修改**/etc/sysctl.conf**文件，需要修改的内容如下。
```conf
# net.ipv4.ip_forward = 0
net.ipv4.ip_forward = 1

# net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.default.rp_filter = 0
```

为了避免不必要的操作失误，不推荐直接修改配置项，可以选择直接覆盖已有配置项的值。
```conf
# 在文件尾部添加下如下内容
# added for xl2tpd
net.ipv4.ip_forward = 1
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.log_martians = 0
net.ipv4.conf.default.log_martians = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.icmp_ignore_bogus_error_responses = 1
```
完成修改后，执行`sysctl -p`指令，使修改生效。

## 中期测试与调整
工作进行到了一半，检验成果的时刻到来了。执行
```bash
# 如果已经开始ipsec，可以执行 service ipsec restart
service ipsec start
ipsec verify
```
操作返回的结果如下，即可视为成功。
```bash
Checking your system to see if IPsec got installed and started correctly:
Version check and ipsec on-path                                 [OK]
Linux Openswan U2.6.24/K2.6.32.16-linode28 (netkey)
Checking for IPsec support in kernel                            [OK]
NETKEY detected, testing for disabled ICMP send_redirects       [OK]
NETKEY detected, testing for disabled ICMP accept_redirects     [OK]
Checking for RSA private key (/etc/ipsec.secrets)               [OK]
Checking that pluto is running                                  [OK]
Pluto listening for IKE on udp 500                              [OK]
Pluto listening for NAT-T on udp 4500                           [OK]
Two or more interfaces found, checking IP forwarding            [OK]
Checking NAT and MASQUERADEing
Checking for 'ip' command                                       [OK]
Checking for 'iptables' command                                 [OK]
Opportunistic Encryption Support                                [DISABLED]
```

### 常见问题
##### SAref kernel support                                  [N/A]
修改**/etc/xl2tpd/xl2tpd.conf**文件如下。
```conf
; ipsec saref = yes
ipsec saref = no
```
更改完成后，不会影响verify的返回结果，但是已可以无视此问题。

##### Two or more interfaces found, checking IP forwarding      [FAILED]
只要`cat /proc/sys/net/ipv4/ip_forward`返回结果是`1`就没事。

> 更多问题，参考[此教程](http://blog.jobbole.com/24004/)

## 继续配置xL2TPD

#### xl2tpd.conf
编辑**/etc/xl2tpd/xl2tpd.conf**文件。
```conf
; 建议直接采用下方指定的IP Range
ip range = 192.168.7.128-192.168.7.254
local ip = 192.168.7.1
```

#### options.xl2tpd
编辑**/etc/ppp/options.xl2tpd**文件，确保整个文件内容如下，注释除外。
```conf
require-mschap-v2
ipcp-accept-local
ipcp-accept-remote
ms-dns 8.8.4.4
ms-dns 8.8.8.8
noccp
auth
crtscts
idle 1800
mtu 1410
mru 1410
nodefaultroute
debug
lock
proxyarp
connect-delay 5000
```

#### 添加L2TP VPN用户
修改**/etc/ppp/chap-secrets**文件，一行代表一个用户。
```conf
# Secrets for authentication using CHAP
# client        server  secret                  IP addresses
name1           *       pass1                   *
name2           *       pass2                   *
```

#### 启动xL2TPD

```bash
service xl2tpd start
```

## 配置iptables
做完上面这堆步骤之后，客户端建个连接就可以验证进入vpn主机了。但是无法访问内外网。执行如下指令，记得替换`$VPS_IP`,此外`192.168.7.0/24`应对应如上**/etc/xl2tpd/xl2tpd.conf**文件中的*IP RANGE*配置。
```bash
iptables -t nat -A POSTROUTING -s 192.168.7.0/24 -o eth0 -j MASQUERADE 
iptables -t nat -A POSTROUTING -s 192.168.7.0/24 -j SNAT --to-source $VPS_IP

iptables-save > /etc/sysconfig/iptables

service ipsec restart
service xl2tpd restart
service iptables restart
```

大功告成！:smile:
