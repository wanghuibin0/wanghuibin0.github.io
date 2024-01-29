# 利用ssh reverse tunnel从远程向本机复制文本


软件开发工作中，用ssh登录后在远程机器上操作，常常需要从远程机器上往本地复制文本。如果是小段的文本，一般在终端选中即可在本地粘贴；但是如果是大段的文本（超过一个屏幕），就没法一次性选中，复制操作变得很麻烦。这里提供一种方法，利用ssh reverse tunnel，可以很方便地将远程大段文本拷贝到本地。

基本思路是：在本地开启一个服务，其监听端口为5689，这个服务持续地将接收到的文本拷贝到剪贴板；利用ssh reverse tunnel将远程机器上的5689端口与本地的5689端口连接起来；在远程机器上，将需要复制的大段文本发送到其5689端口，这段文本就会出现在本地机器的剪贴板里。

## 本地开启服务
命令行中执行以下代码：
```bash
while (true); do nc -l -p 5689 | xclip -selection c; done
```
启动netcat监听5689端口，将接收到的任何数据拷贝到剪贴板，然后继续监听。

现在，可以在本地测试一下效果：
```bash
echo "send me to local clipboard" | nc -q0 localhost 5689
# -q0 是为了去除文本末尾的eof
```
## 将服务暴露给远程机器
ssh reverse tunnel 可以在远程机器和本地机器之间建立一个通道，在远程机器和本地机器各指定一个端口，并将它们连接在一起，这是通过-R参数实现的。
```bash
ssh user@remote-server.com -R 5689:localhost:5689
```
也可以将此配置写入`.ssh/config`，在对应的host条目里添加下面这行，以后登录ssh都会自动建立这个通道。
```bash
RemoteForward 5689 localhost:5689
```
现在，可以在远程机器上执行以下命令测试一下效果：
```bash
echo "send me to local clipboard" | nc -q0 localhost 5689
```
## 从远程机器发送文本
经过以上步骤，只要在远程机器上将数据发往其5689端口，数据就会出现在本地剪贴板了。不过我最常用的场景是在vim中复制文本，把这部分操作进一步简化就很有必要了。为此，我在.vimrc中添加如下代码：
```vim
" :CL copy from remote ssh to local
command! -range CL execute '<line1>,<line2>w !nc -N localhost 5689'
```
添加了一个自定义命令CL，执行这个命令时，被选中的文本就会发送到5689端口，进而传回到本机的剪贴板。

如此，当我在远程机器上的vim中想要复制文本时，我就先选中这段文本，然后执行命令`:CL`，接下来就可以从本地机器的剪贴板粘贴了。

## 更进一步
在远程机器的.bashrc中设置以下命令：
```bash
ncp() {
  nc -N localhost 5689
}

cp2l() {
  base64 $1 | ncp
}
```
在本地机器的.bashrc中设置以下命令：
```bash
function cp2l {
  xclip -selection c -o | base64 -d > $1
}
```

这样，分别在远程和本地机器上执行以下命令（会利用base64工具的编码和解码功能）：
```bash
cp2l file
```
就可以将远程机器上的file文件复制到本地了。

## 参考
- https://gist.github.com/dergachev/8259104
- https://qbee.io/misc/reverse-ssh-tunneling-the-ultimate-guide/

