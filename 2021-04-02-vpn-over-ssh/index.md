# VPN over SSH---利用SSH进行简易科学上网

今天在浏览《科技爱好者周刊》时，学会了一个神奇的技能：VPN over SSH，直接利用ssh搭建一个简易的VPN。虽然现在有了V2ray/Xray/Trojan/SSR等各种代理工具，但是这些都需要额外的步骤安装部署，如果只是临时使用的话，远远没有VPN over SSH简单方便，因为SSH在PC和服务器上基本都是标配了。
<!--more-->

话不多说，直接开始。

### 创建SOCKS代理

> $ ssh -NTCD 12345 username@SSH_remote_host_IP
> =====================================
>
> -N Do not execute a remote command
>
> -T Disable pseudo-terminal allocation
> -C Requests compression of all data
> -D \<port>
> Specifies a local "dynamic" application-level port forwarding. This works by allocating a socket to listen to the port on the local side, optionally Whenever a connection is made to this port, the connection is forwarded over the secure channel, and the application protocol is then used to determine where to connect to from the remote machine. Currently the SOCKS4 and SOCKS5 protocols are supported, and ssh will act as a SOCKS server.
>

上述命令直接在本机创建了一个SOCKS服务器，监听端口为12345。

### 浏览器使用SOCKS代理

可以在internet选项->连接里面修改，也可以利用Proxy SwitchyOmega等插件。

### 其他软件使用SOCKS代理

SOCKS是比较低层的协议，如果按照OSI模型来讲的话，SOCKS应该属于会话层，所以更高层的协议如FTP/HTTP/SSH等都可以利用SOCK5代理通信。

#### SSH

特别简单，只需要一条命令：

`ssh -o ProxyCommand='nc -x localhost:12345 %h %p' username@**Far_Away_Host**`

#### 其他

可针对具体软件具体设置。

### sshuttle

sshuttle也是一个基于ssh实现vpn的工具，使用也很简单：`$ sshuttle -r username**@SSH_remote_host_IP** 0.0.0.0/0`，就可以将本机上所有流量转发到远程vps上去，实现代理通信，就不用再手动进行各软件的代理设置了。不过个人认为，对于简单的需求，临时vpn代理，单纯的ssh就可以了，要是有更复杂的需求，还是用xray这种专门的科学上网工具好了，sshuttle反而没太大必要。

### 参考

[VPN over SSH? The SOCKS Proxy](https://blog.gwlab.page/vpn-over-ssh-the-socks-proxy-8a8d7bdc7028)

