---
title: OpenVpn4Centos
comments: true
date: 2017-07-11 15:15:14
updated: 2017-07-11 15:15:14
tags:
- OpenVpn
categories:
- OpenVpn
---
### 声明
- OpenVpn版本: 2.4.3  
- Centos版本: 6.8  
- 本机IP: 192.168.5.3
- 实现功能：
  - 客户端需通过用户名密码登录
  - 用户名密码为系统用户
  - 不同用户分为不同网段
  - 通过VPN服务器的iptables来限制不同网段的访问，间接限制用户
先实现证书登录，再实现用户名密码登录，也可以跳过证书登录的配置，直接实现用户名密码登录
<!--more-->

| 用户     | 获得网段 | 可访问网段                                           |
|:---------|:----------------------------------------------------------------|
| 普通用户 | 10.0.2.0/24   |   192.168.3.0/24 192.168.4.0/24  192.168.5.0/24 |
| 特殊用户 | 10.0.1.0/24 | 192.168.0.0/24 192.168.1.0/24 192.168.5.0/24      |
| 管理员   | 10.0.0.0/24 | 192.168.0.0/16                                    |

### 安装
#### 安装epel库
```
wget http://dl.fedoraproject.org/pub/epel/6/i386/epel-release-6-8.noarch.rpm
rpm -Uvh epel-release-6-8.noarch.rpm
```

#### 安装openvpn
```
yum install openvpn
```
#### 安装easy-rs
该包用来制作ca证书，服务端证书，客户端证书。最新的为easy-rsa3
```
wget https://github.com/OpenVPN/easy-rsa/archive/master.zip
unzip master.zip
mv easy-rsa-mater/ easy-rsa/
cp -R easy-rsa/ /etc/openvpn/
```

#### 编辑vars文件
1. 先进入/etc/openvpn/easy-rsa/easyrsa3目录
```
cd /etc/openvpn/easy-rsa/easyrsa3/
```

2. 复制vars.example 为vars
```
cp vars.example vars
```

3. 修改vars文件的如下字段
```
set_var EASYRSA_REQ_COUNTRY “CN” //根据自己情况更改
set_var EASYRSA_REQ_PROVINCE “ShangHai”
set_var EASYRSA_REQ_CITY “ShangHai”
set_var EASYRSA_REQ_ORG “Test”
set_var EASYRSA_REQ_EMAIL “test@test.com”
set_var EASYRSA_REQ_OU “TestOpenVpn”
```

#### 初始化easyrsa
```
cd /etc/openvpn/easy-rsa/easyrsa3/
./easyrsa init-pki
```

#### 创建CA(根)证书
```
./easyrsa build-ca

Generating a 2048 bit RSA private key
…………………………………….+++
……+++

writing new private key to ‘/root/easy-rsa/easyrsa3/pki/private/ca.key’
Enter PEM pass phrase:
Verifying – Enter PEM pass phrase:
—–
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter ‘.’, the field will be left blank.
—–
Common Name (eg: your user, host, or server name) [Easy-RSA CA]:Test
CA creation complete and you may now import and sign cert requests.
Your new CA certificate file for publishing is at:
/root/easy-rsa/easyrsa3/pki/ca.crt
```
注意：在上述部分需要输入PEM密码 PEM pass phrase，输入两次，此密码必须记住，
不然以后不能为证书签名。还需要输入common name 通用名，这个你自己随便设置个独一无二的。

#### 创建服务器端证书
```
./easyrsa gen-req server nopass


Generating a 2048 bit RSA private key
……………………………………………………………………..+++
……………………+++
writing new private key to ‘/root/easy-rsa/easyrsa3/pki/private/server.key’
—–
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter ‘.’, the field will be left blank.
—–
Common Name (eg: your user, host, or server name) [server]:TestServer
Keypair and certificate request completed. Your files are:
req: /root/easy-rsa/easyrsa3/pki/reqs/server.req
key: /root/easy-rsa/easyrsa3/pki/private/server.key
```
该过程中需要输入common name，随意但是不要跟之前的根证书的一样

#### 签约服务端证书
```
./easyrsa sign server server


You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.
Request subject, to be signed as a server certificate for 3650 days:
subject=
commonName = TestServer
Type the word ‘yes’ to continue, or any other input to abort.
Confirm request details: yes
Using configuration from /root/easy-rsa/easyrsa3/openssl-1.0.cnf
Enter pass phrase for /root/easy-rsa/easyrsa3/pki/private/ca.key:
Check that the request matches the signature
Signature ok
The Subject’s Distinguished Name is as follows
commonName
RINTABLE:’TestServer’
Certificate is to be certified until Apr 20 06:02:10 2024 GMT (3650 days)
Write out database with 1 new entries
Data Base Updated
Certificate created at: /root/easy-rsa/easyrsa3/pki/issued/server.crt
```
该命令中.需要你确认生成，要输入yes，还需要你提供我们当时创建CA时候的密码。如果你忘记了密码，
那你就重头开始再来一次吧。

#### 创建Diffie-Hellman
```
./easyrsa gen-dh


Note: using Easy-RSA configuration from: ./vars
Generating DH parameters, 2048 bit long safe prime, generator 2
This is going to take a long time
……..+……………………………….+..+…………………………………………………………………………………………………………………………….
DH parameters of size 2048 created at /etc/openvpn/easy-rsa/easyrsa3/pki/dh.pem
```

#### 创建客户端证书
以下是为了实现证书登录而做的操作。  
为了避免与服务器端证书冲突，因此另外复制一份easyrsa到另一个文件夹下操作。  
进入root目录新建client文件夹(也可以是任意目录，只要有权限访问)，文件夹可随意命名，
然后拷贝前面解压得到的easy-ras文件夹到client文件夹,进入下列目录,并初始化
```
cd /root/
mkdir client
cp -R easy-rsa/ client/
cd client/easy-rsa/easyrsa3/
./easyrsa init-pki
```
- 创建客户端key及生成证书（记住生成是自己输入的密码）
```
./easyrsa gen-req client (名字自定义)
```

- 将的到的client.req导入然后签约证书
```
cd /etc/openvpn/easy-rsa/easyrsa3/
./easyrsa import-req /root/client/easy-rsa/easyrsa3/pki/reqs/client.req client
./easyrsa sign client client(后面这个为客户端证书的名字，前面的为固定格式)
```
过程跟签约server类似，需要输入CA证书的密码

#### 现在说一下我们上面都生成了什么东西
服务端：(/cetc/openvpn/easy-rsa/文件夹)
- /etc/openvpn/easy-rsa/easyrsa3/pki/ca.crt
- /etc/openvpn/easy-rsa/easyrsa3/pki/reqs/server.req
- /etc/openvpn/easy-rsa/easyrsa3/pki/reqs/client.req
- /etc/openvpn/easy-rsa/easyrsa3/pki/private/ca.key
- /etc/openvpn/easy-rsa/easyrsa3/pki/private/server.key
- /etc/openvpn/easy-rsa/easyrsa3/pki/issued/server.crt
- /etc/openvpn/easy-rsa/easyrsa3/pki/issued/client.crt
- /etc/openvpn/easy-rsa/easyrsa3/pki/dh.pem

客户端：(root/client/easy-rsa文件夹)
- /root/client/easy-rsa/easyrsa3/pki/private/client.key
- /root/client/easy-rsa/easyrsa3/pki/reqs/client.req //这个文件被我们导入到了服务端,
                                                       所以那里也有
#### 将生成的证书拷贝到指定文件夹(/etc/openvpn/keys)
```
cp /etc/openvpn/easy-rsa/easyrsa3/pki/ca.crt /etc/openvpn/keys
cp /etc/openvpn/easy-rsa/easyrsa3/pki/private/server.key /etc/openvpn/keys
cp /etc/openvpn/easy-rsa/easyrsa3/pki/issued/server.crt /etc/openvpn/keys
cp /etc/openvpn/easy-rsa/easyrsa3/pki/dh.pem /etc/openvpn/keys
```

#### 生成ta.key文件并移动到/etc/openvpn/keys
```
openvpn --genkey --secret ta.key
mv ta.key /etc/openvpn/keys
```

#### 编辑server的配置文件
当你安装好了openvpn时候，他会提供一个server配置的文件例子，在
/usr/share/doc/openvpn-2.4.3/sample/sample-config-files
下会有一个server.conf文件，我们将这个文件复制到/etc/openvpn
```
cp /usr/share/doc/openvpn-2.4.3/sample/sample-config-files/server.conf /etc/openvpn
```
修改配置文件为如下内容

```
local 192.168.5.3 #本地IP，既服务器的IP地址
port 1194 #vpn端口
proto udp #使用UDP协议，也可以选择tcp协议
dev tun #相对应的tab，tab是桥接模式，tun为虚拟网卡模式
ca /etc/openvpn/keys/ca.crt #ca证书
cert /etc/openvpn/keys/server.crt #服务器端证书
key /etc/openvpn/keys/server.key  #服务器端的key，需保密
dh /etc/openvpn/keys/dh.pem #dh证书
server 10.0.2.0 255.255.255.0 #普通vpn用户虚拟IP的网段
ifconfig-pool-persist /etc/openvpn/ipp.txt #虚拟IP记录文件，防止重复分发IP
push "route 192.168.0.0 255.255.255.0" #网客户端注入route规则，用来实现部分网段走vpn，
                                        其他网段仍然走客户端电脑的默认链路
push "route 192.168.1.0 255.255.255.0"
push "route 192.168.2.0 255.255.255.0"
push "route 192.168.3.0 255.255.255.0"
push "route 192.168.4.0 255.255.255.0"
push "route 192.168.5.0 255.255.255.0"
client-config-dir /etc/openvpn/ccd #用户个性化配置目录，特殊用户与管理员用户
                                    的网段就是通过这个文件夹下的配置实现的
route 10.0.0.0 255.255.255.0 #添加本地route，将特殊用户与管理员用户
                              的网段加入vpn服务器的路由表中
route 10.0.1.0 255.255.255.0
push "dhcp-option DNS 114.114.114.114" #为客户端电脑得到的虚拟IP推送DNS
duplicate-cn #允许同一个证书或同一个用户同一时间多次登录
keepalive 10 120 #客户端与服务器的心跳包，互相知道对方是否断开
tls-auth /etc/openvpn/keys/ta.key 0 #ta证书，需保密
cipher AES-256-CBC #加密规则
comp-lzo #兼容旧的客户端
max-clients 100 #客户端数量
persist-key
persist-tun
status /etc/openvpn/logs/openvpn-status.log #日志文件
log         /etc/openvpn/logs/openvpn.log #日志文件
verb 3 #日志等级
```
创建 /etc/openvpn/logs文件夹
创建 /etc/openvpn/ccd文件夹

#### 配置特殊用户的个性化配置
文件名必须与客户端证书名或者与下面即将配置的用户名密码登录的用户名一致，
一个用户一个对应的文件，如果没有，则按默认配置  
这里假设client为特殊用户，既获得的网段在192.168.1.0/24。  
需要创建client对应的客户端证书(如果使用用户名密码登录，需要创建对应用户)
```
touch client
vi client

ifconfig-push 10.0.1.1 10.0.1.2
# 这里是指定该用户的IP，既直接固定IP，而不是随机分配。暂时没找到指定IP段的方法。
因此，通过固定IP的方式来达到特殊用户属于特殊网段
# 这里指定的是一对IP，表示虚拟客户端和服务器的IP端点。
它们必须从连续的/30子网网段中获取
(这里是/30表示xxx.xxx.xxx.xxx/30，即子网掩码位数为30)，
以便于与Windows客户端和TAP-Windows驱动兼容。暂时还没理解。
明确地说，每个端点的IP地址对的最后8位字节必须取自下面的集合：
[  1,  2]  [  5,  6]  [  9, 10]  [ 13, 14]
[ 17, 18]  [ 21, 22]  [ 25, 26]  [ 29, 30]   
[ 33, 34]  [ 37, 38]  [ 41, 42]  [ 45, 46]
[ 49, 50]  [ 53, 54]  [ 57, 58]  [ 61, 62]   
[ 65, 66]  [ 69, 70]  [ 73, 74]  [ 77, 78]
[ 81, 82]  [ 85, 86]  [ 89, 90]  [ 93, 94]   
[ 97, 98]  [101,102]  [105,106]  [109,110]   
[113,114]  [117,118]  [121,122]  [125,126]   
[129,130]  [133,134]  [137,138]  [141,142]   
[145,146]  [149,150]  [153,154]  [157,158]
[161,162]  [165,166]  [169,170]  [173,174]   
[177,178]  [181,182]  [185,186]  [189,190]   
[193,194]  [197,198]  [201,202]  [205,206]   
[209,210]  [213,214]  [217,218]  [221,222]   
[225,226]  [229,230]  [233,234]  [237,238]
[241,242]  [245,246]  [249,250]  [253,254]
```

#### 下载openvpn客户端，并进行配置
用sftp将我们在vpn服务器上生成的客户端证书和key下载到客户端电脑，包括如下四个文件
- /etc/openvpn/easy-rsa/easyrsa3/pki/ca.crt
- /etc/openvpn/easy-rsa/easyrsa3/pki/issued/client.crt
- /root/client/easy-rsa/easyrsa3/pki/private/client.key
- /etc/openvpn/keys/ta.key
去官网下载openvpn客户端进行安装，然后安装目录找到simple-config，
默认为C:\Program Files\OpenVPN\sample-config\client.ovpn。  
将client.ovpn 复制到openvpn的config目录下，
默认为C:\Program Files\OpenVPN\config
将下载到的四个文件同样放入config目录下，
默认为C:\Program Files\OpenVPN\config  

修改配置文件为如下内容：
```
client #标记为客户端
dev tun #与服务器端配置一致
proto udp #与服务器端配置一致
remote 122.112.219.248 1194 #服务器端IP与端口
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt #ca证书
cert client.crt #客户端证书
key client.key #客户端证书的key
remote-cert-tls server #服务器证书的名字
tls-auth ta.key 1 #ta证书，如果服务器端配置，则客户端必须配置
cipher AES-256-CBC #与服务器端配置一致
comp-lzo #与服务器端配置一致
verb 3 #日志级别
```

经过以上配置，客户端就能连接上vpn服务器了(服务器端的1194端口要开放)，但是并不能访问内网服务器，
因为vpn服务器没有配置虚拟ip的转发。

#### 配置服务器端iptables
通过如下命令来达到将某个网段的IP转发到主网卡上，以达到与内网服务器通信的目的。
```
iptables -t nat -A POSTROUTING -s 10.0.1.0/24 -d 192.168.0.0/24 -o eth0 -j MASQUERADE
```
为了实现不同用户不同权限，需要做如下操作来限制不同网段的用户的访问权限
```
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 10.0.1.0/24 -d 192.168.0.0/24 -o eth0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 10.0.1.0/24 -d 192.168.1.0/24 -o eth0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 10.0.1.0/24 -d 192.168.5.0/24 -o eth0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 10.0.2.0/24 -d 192.168.3.0/24 -o eth0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 10.0.2.0/24 -d 192.168.4.0/24 -o eth0 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 10.0.2.0/24 -d 192.168.5.0/24 -o eth0 -j MASQUERADE
```
经过如此配置，即可访问内网
配置完成后，使用如下命令保存iptables 规则到文件
```
service iptables save
```
#### 实现用户名密码登录
在经过以上配置后，再来实现用户名密码登录就简单多了，只需要改一些配置即可。

##### 修改服务器端配置
编辑/etc/openvpn/server.conf,添加如下内容
```
plugin /usr/lib64/openvpn/plugin/lib/openvpn-auth-pam.so login #我是64位操作系统
client-cert-not-required #只需验证用户名密码，不要求客户端证书
username-as-common-name #用户名做common-name，既用户名相当于客户端名，个性化的时候使用用户名即可。
```

编辑客户端配置文件，既C:\Program Files\OpenVPN\sample-config\client.ovpn
注释掉如下内容
```
;cert client.crt
;key client.key
```
添加如下内容
```
auth-user-pass
```

创建服务器用户，由于仅作为vpn用户登录，因此建议创建不带home目录，没有登录权限的用户
```
useradd -M user1 -s /sbin/nologin
echo "password" | passwd user1 --stdin
```
