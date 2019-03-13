# OpenVPN for Docker

[![Build Status](https://travis-ci.org/kylemanna/docker-openvpn.svg)](https://travis-ci.org/kylemanna/docker-openvpn)
[![Docker Stars](https://img.shields.io/docker/stars/kylemanna/openvpn.svg)](https://hub.docker.com/r/kylemanna/openvpn/)
[![Docker Pulls](https://img.shields.io/docker/pulls/kylemanna/openvpn.svg)](https://hub.docker.com/r/kylemanna/openvpn/)
[![ImageLayers](https://images.microbadger.com/badges/image/kylemanna/openvpn.svg)](https://microbadger.com/#/images/kylemanna/openvpn)
[![FOSSA Status](https://app.fossa.io/api/projects/git%2Bgithub.com%2Fkylemanna%2Fdocker-openvpn.svg?type=shield)](https://app.fossa.io/projects/git%2Bgithub.com%2Fkylemanna%2Fdocker-openvpn?ref=badge_shield)
[![Anchore Image Overview](https://anchore.io/service/badges/image/af41b351247fc340958e9c67aed342860da328339257f809c043c865679d981d)](https://anchore.io/image/dockerhub/kylemanna%2Fopenvpn%3Alatest)


本项目是带有EasyRSA PKI CA的OpenVPN Server Docker容器.

本镜像在在[Digital Ocean $5/mo node](http://bit.ly/1C7cKr3)节点上进行了广泛测试，并拥有相应的数字海洋社区教程。

#### 上游链接

* Docker仓库 @ [kylemanna/openvpn](https://hub.docker.com/r/kylemanna/openvpn/)
* GitHub @ [kylemanna/docker-openvpn](https://github.com/kylemanna/docker-openvpn)

## 快速入门

* 为变量`$OVPN_DATA`设置一个数据卷容器名称。建议使用`ovpn-data-`前缀以实现与systemd服务间无缝地操作。建议用户使用他们选择的名称替换`example`。

      OVPN_DATA="ovpn-data-example"

* 初始化用以保存配置文件和证书的`$OVPN_DATA`容器。容器将提示输入密码以保护生成的CA证书私钥。 

      docker volume create --name $OVPN_DATA
      docker run -v $OVPN_DATA:/etc/openvpn --log-driver=none --rm kylemanna/openvpn ovpn_genconfig -u udp://VPN.SERVERNAME.COM
      docker run -v $OVPN_DATA:/etc/openvpn --log-driver=none --rm -it kylemanna/openvpn ovpn_initpki

* 启动OpenVPN服务器进程

      docker run -v $OVPN_DATA:/etc/openvpn -d -p 1194:1194/udp --cap-add=NET_ADMIN kylemanna/openvpn

* 创建无密码的客户端证书

      docker run -v $OVPN_DATA:/etc/openvpn --log-driver=none --rm -it kylemanna/openvpn easyrsa build-client-full CLIENTNAME nopass

* 获取嵌入客户端证书的客户端配置文件

      docker run -v $OVPN_DATA:/etc/openvpn --log-driver=none --rm kylemanna/openvpn ovpn_getclient CLIENTNAME > CLIENTNAME.ovpn

## 下一步

### 扩展阅读

[docs](docs) 文件夹中提供了高级配置说明文件。

### Systemd初始化脚本

`systemd`初始化脚本可用于管理OpenVPN容器。它将在系统启动时启动容器，
如果容器意外退出则重新启动容器，并从Docker Hub获取更新以使容器内程序保持最新。

请参阅[systemd documentation](docs/systemd.md)文档以了解更多信息。

### Docker Compose

如果您更喜欢使用`docker-compose`，请参阅[documentation](docs/docker-compose.md)。

## 调试技巧

* 创建名为DEBUG且值为1的环境变量以启用调试输出（使用"docker -e"）。

        docker run -v $OVPN_DATA:/etc/openvpn -p 1194:1194/udp --privileged -e DEBUG=1 kylemanna/openvpn

* 正确安装openvpn客户端后进行测试

        $ openvpn --config CLIENTNAME.ovpn

* 如果测试失败，请在客户端主机上进行一些测试以确保网络是正确的（中国国内用户需要修改下面三参数后进行测试）

        $ ping 8.8.8.8    # checks connectivity without touching name resolution
        $ dig google.com  # won't use the search directives in resolv.conf
        $ nslookup google.com # will use search

* 考虑设置一个[systemd service](/docs/systemd.md)服务，以便在系统启动时自动启动容器，
  并在OpenVPN守护程序或Docker崩溃的情况下重新启动。

## 工作原理

使用包含脚本的`kylemanna/openvpn`镜像初始化卷容器以便自动生成：

- Diffie-Hellman 参数
- 一个私钥（private key）
- 一个与OpenVPN服务器的私钥匹配的自签证书
- 一个EasyRSA CA 密钥和证书
- 一个来自HMAC安全的TLS认证密钥

OpenVPN服务器默认以`ovpn_run`命令启动

配置文件位在目录`/etc/openvpn`中，Dockerfile将该目录声明为卷。
这意味着您可以使用`-v`参数启动另一个容器，并访问该配置。
该卷还包含PKI密钥和证书，以便可以备份它。

要生成客户端证书，`kylemanna/openvpn`通过容器中的`easyrsa`命令使用EasyRSA。 
`EASYRSA_*`环境变量将PKI CA放在`/etc/openvpn/pki`路径下。

幸运的是，`kylemanna/openvpn`附带了一个名为`ovpn_getclient`的脚本，
该脚本会生成与OpenVPN服务器相匹配的OpenVPN客户端配置文件。然后将该单个文件提供给客户端以访问VPN。

要为客户端启用双因子身份验证(a.k.a. OTP)，请参阅此[this document](/docs/otp.md)。

## OpenVPN细节

我们使用适用于最广泛设备的`tun`模式。例如`tap`不适用于Android系统，除非Android系统已经ROOT过。

使用的拓扑结构是`net30`，因为它适用于最广泛的操作系统。例如`p2p`在Windows上不起作用。

默认情况下，UDP服务器对动态客户端使用`192.168.255.0/24`网段设置。

客户端配置文件指定`redirect-gateway def1`，这意味着在建立VPN连接后，所有流量都将通过VPN。
这有可能导致您使用无法直接访问您的本地DNS服务器。如果发生这种情况，
请使用Google（8.8.4.4和8.8.8.8）或OpenDNS（208.67.222.222和208.67.220.220）等公共DNS解析器。


## 安全讨论

Docker容器运行自己的EasyRSA PKI证书颁发系统。这样选择是在安全性和便利性上妥协的好方法。
OpenVPN容器在确保安全的主机上运行，也就是说攻击者无法访问`/etc/openvpn/pki`下的PKI文件。
这是一个合理的折衷方案，因为如果假定攻击者可以访问这些文件，那攻击者其实可以完全操纵OpenVPN服
务器本身的功能 (sniff packets, create a new PKI CA, MITM packets, etc)。

* 为简单起见，默认情况下，证书颁发机构密钥保留在容器中。强烈建议使用密码保护CA密钥以防止文件系统被攻击。
  更安全的系统会将EasyRSA PKI CA置于离线系统上（可以使用相同的Docker镜像和脚本
  [`ovpn_copy_server_files`](/docs/paranoid.md)来实现此目的）。
* 如果攻击者取得了文件系统的root访问权限，那么如果没有先破解密钥的密码，攻击者就不可能签署错误或伪造的证书。
* EasyRSA `build-client-full`命令将在服务器上生成并保留密钥，再次可能危及和窃取密钥。
  生成的密钥需要由上术密码保护的CA进行签名。
* 假定Docker容器的其余文件系统是安全的，TLS + PKI安全性应该可以防止任何恶意主机使用VPN。


## 在Docker容器内运行的好处

### 整个守护程序和依赖项都在Docker镜像中

这意味着它在所有Linux发行版上都能正常运行（需要安装Docker），例如：Ubuntu，Arch，Debian，Fedora等。
此外，旧的稳定服务器可以运行最新的OpenVPN服务器而无需安装/弄糟操作系统的依赖库（例如在Ubuntu 12.04 LTS
上运行最新的OpenVPN和最新的OpenSSL）。

### 它不会污染宿主服务器的文件系统

Docker容器的所有内容都包含在两个镜像中：运行时映像（kylemanna/openvpn）和`$OVPN_DATA`数据卷。
要删除它，只需删除相应的容器，`$OVPN_DATA`数据卷和Docker镜像即可。这也使得运行多个服务器变得更容易，
因为每个服务器都存在于容器的泡中（当然，需要多个IP或单独的端口来与外部通信）。

### 一些（可探讨）安全优势

简单来说，即使容器被人攻破，还可以防止宿主服务器的被攻破。关于这个问题有很多争议，但要注意的是，它肯定会使得攻破容器
变得更加困难。人们正在积极研究Linux容器，以便在未来能更好地保证安全。

## 与jpetazzo/dockvpn的区别

* 不再使用serveconfig通过https分发配置文件
* 适度的PKI支持集成到镜像像中
* OpenVPN配置文件，PKI密钥和证书存储在存储卷上，以便跨容器重用
* 为HMAC安全添加tls-auth

## 测试过的平台

* Docker主机:
  * [Digital Ocean](https://www.digitalocean.com/?refcode=d19f7fe88c94)Droplet服务器，带512 MB 内存，运行Ubuntu 14.04 server操作系统
* 客户端
  * Android App OpenVPN Connect 1.1.14 (built 56)
     * OpenVPN core 3.0 android armv7a thumb2 32-bit
  * OS X Mavericks with Tunnelblick 3.4beta26 (build 3828) using openvpn-2.3.4
  * ArchLinux OpenVPN pkg 2.3.4-1


## License
[![FOSSA Status](https://app.fossa.io/api/projects/git%2Bgithub.com%2Fkylemanna%2Fdocker-openvpn.svg?type=large)](https://app.fossa.io/projects/git%2Bgithub.com%2Fkylemanna%2Fdocker-openvpn?ref=badge_large)
