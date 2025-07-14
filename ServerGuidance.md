# **iData 集群服务器操作指南**

**本指南旨在为用户提供 iData 集群服务器的详细操作步骤和注意事项。**

快速跳转查看各章节：

[TOC]

------

## 一、服务器概览

**服务器物理位置:** 所有服务器位于3号楼2楼207实验室，进门右转玻璃机房内。机架后方提供显示器和键鼠用于操作跳板机。

|    设备    |    IP地址     |      配置       |               备注               |
| :--------: | :-----------: | :-------------: | :------------------------------: |
|   跳板机   | 10.19.126.194 |        /        |        dhcp，ssh用该地址         |
| 跳板机内网 | 192.168.0.100 |        /        | 计算节点反过来操作跳板机用该地址 |
|   节点1    | 192.168.0.101 |   RTX 1080 Ti   |      nvidia-smi option无效       |
|   节点3    | 192.168.0.103 |  RTX 3080 * 4   |                                  |
|   节点4    | 192.168.0.104 |   RTX 1080 Ti   | 驱动好像有问题，找不到nvidia-smi |
|   节点5    | 192.168.0.105 | RTX 1080 Ti * 3 |                                  |
|   节点6    | 192.168.0.106 |  RTX 3080 * 4   |                                  |

------

## 二、网络连接与认证

### 2.1 校外连接

校外用户需先使用学校提供的 VPN 连接到校园网。

### 2.2 跳板机网络认证

1. 联网原理：跳板机做http代理让计算节点联网，配置方法见**squid**代理部分
2. 首先在**207实验室-服务器机房**内登录跳板机系统：***用户名：root；密码：Dell.com 或 root***
3. 系统跳板机使用 DHCP 模式上网，并通过学生账号进行认证，认证方式与 Wi-Fi 认证相同，一般网线连接后会自动跳出认证界面。认证地址:https://net-auth.shanghaitech.edu.cn:19008/protalauth/login，或者https://net-auth.shanghaitech.edu.cn:19008/portalpage

### 2.3 问题解决

1. **机房内找不到跳板机？** 跳板机位于机房靠墙一侧，机器相对于服务器节点机器来说较扁。
2. **跳板机上认证界面打不开？** 尝试重启squid代理。
3. **其他问题？** 其他网络问题，请联系组内学长学姐或学院技术人员。学校信息服务网址：[上海科技大学信息化服务网站](https://it.shanghaitech.edu.cn/)；服务热线：***021-20685566***；服务邮件：***it-support@shanghaitech.edu.cn***

------

## 三、SSH 使用指南

### 3.1 VSCode使用示例

1. 安装插件 **remote-ssh**

2. 点击远程资源管理器，打开SSH配置文件，点击第一个路径的config文件

3. 填入以下代码并保存

   ```
   #SSH配置
   Host JumpMachine             #跳板机名称
   	HostName 10.19.126.194   #跳板机IP 
   	User root                #跳板机用户名
   
   Host TargetMachine103           #远程服务器名称
   	HostName 192.168.0.103 #远程服务器IP
   	User root                #远程服务器用户名
   	ProxyCommand ssh -W %h:%p JumpMachine
   
   Host TargetMachine106           #远程服务器名称
   	HostName 192.168.0.106 #远程服务器IP
   	User root                #远程服务器用户名
   	ProxyCommand ssh -W %h:%p JumpMachine
   ```

4. 在VSCode的远程资源管理器中进行连接。密码：***Dell.com***

5. 假如提示The authenticity of host <host> can't be established，这说明你的电脑首次ssh到该host，可以输入yes后回车

### 3.2 文件传输 (SCP)

1. **上传文件：** scp 本地文件名 root@跳板机IP:跳板机的目录
2. **下载文件：** scp root@跳板机IP:跳板机上的文件名 本地目录
3. **上传文件夹：** scp -r 本地文件夹 root@跳板机IP:跳板机的目录
4. **下载文件夹：** scp -r root@跳板机IP:跳板机的文件夹 本地目录
5. **直接传输计算节点文件/文件夹:** 在 scp 命令中添加 -o 'ProxyJump root@跳板机IP' 参数。例如，下载节点5上的文件夹：scp -o 'ProxyJump root@跳板机IP' -r root@192.168.0.105:节点5上的文件夹 本地目录
6. **使用节点别名进行文件传输：**例如节点3名称为TargetMachine103，从节点3下载文件夹：scp -r TargetMachine103:节点3上的文件夹 本地目录

### 3.3 SSH 免密登录 (密钥对认证)

如果想在ssh使用过程中免密码，可以生成密钥对来进行认证，方法是：

在本地windows电脑上打开命令行，输入ssh-keygen

1. **Enter file in which to save the key (C:\Users\windows用户名/.ssh/id_rsa):** 这里可以指定存储密钥的文件名，也可以直接按回车使用默认文件名，默认的文件名是id_rsa（私钥）和id_rsa.pub（公钥）
2. **Enter passphrase (empty for no passphrase):** passphrase是用来加密私钥的，可以不指定，如果指定了passphrase，那么在认证过程中需要输入passphrase来解密私钥，这么做是为了保证即使私钥被窃取（电脑不慎遗失/被入侵），也无法直接通过私钥连入服务器

接下来，需要把生成的公钥写入跳板机和服务器的~/.ssh/authorized_keys文件中
不需要在跳板机上再生成一次密钥，ProxyJump的方式两次认证都是采用的本地电脑的私钥！！！
但假如你的ssh版本不支持ProxyJump，而是用ProxyCommand或者其他方式，那么下面的方法可能并不适用。或许需要在跳板机上再次创建密钥并把跳板机的公钥写入服务器的~/.ssh/authorized_keys文件

如果你不是Windows，那么可以尝试使用：ssh-copy-id -i ~/.ssh/id_rsa.pub root@跳板机IP命令复制公钥
由于Windows 10的ssh配置中没有ssh-copy-id命令，所以需要在powershell中使用：type $env:USERPROFILE\.ssh\id_rsa.pub | ssh root@跳板机IP "cat >> .ssh/authorized_keys"

之后需要再把公钥复制到服务器上，操作方式类似，和之前同理，也可以用.ssh\config文件中指定的别名，或者在ssh-copy-id或ssh命令中加上-o 'ProxyJump root@跳板机IP'选项
如果复制完成后登录还需要密码，请检查一下~/.ssh/authorized_keys是否填写正确，文件中每个公钥都需要单独占一行，可能会出现复制过去没有换行的情况

------

## 四、重要注意事项

### 3.1 MAC 地址封禁

如果服务器无法通过账号密码上网，并提示*“认证失败”*、*“设备认证请求失败”*或*“Radius 服务器认证失败”*，则说明 MAC 地址可能被封禁。

**解封方式:** 需要通过重装系统或全盘杀毒来解封。可以联系学校信息服务：信息服务网址：[上海科技大学信息化服务网站](https://it.shanghaitech.edu.cn/)；服务热线：***021-20685566***；服务邮件：***it-support@shanghaitech.edu.cn***

### 3.2 矿池病毒防范

跳板机无法通过认证联网，可能是被图信误封，但也有感染矿池病毒的可能（学校技术人员说确实有在其他服务器上查到过矿池病毒）。

为了防止跳板机中矿池病毒，应注意以下几点：
1. 不要将跳板机的root用户密码设置为简单密码
2. 防止私钥泄漏，不要使用rsa密钥对免密登录跳板机
3. 当使用conda时，直接使用官方源（下载速度足够快），从国内其他源下载有被攻击的风险
4. 当不需要远程ssh连接时，可以考虑进机房关闭跳板机的外网网口em4（但是这比较麻烦，不强制）

------

## 五、在跳板机上配置 Squid 代理

安装 Squid

```
sudo yum install squid
```

安装squid

```
sudo yum install squid
```

查看squid服务状态

```
systemctl status squid
```

启动服务

```
systemctl start squid
```

把服务设置为开机启动并现在启动

```
systemctl enable --now squid
```

停止服务

```
systemctl stop squid
```

重启服务（在更新配置文件后需要重启）

```
systemctl restart squid
```

假如在操作过程中提示域名解析不成功，那么需要添加dns配置（通常不需要）
dns_nameservers 8.8.8.8 8.8.4.4（8.8.8.8和8.8.4.4是google提供的公共dns服务器）
用vi打开/etc/squid/squid.conf，写入上面这一行

永久性添加防火墙规则并重启防火墙

```
firewall-cmd --permanent --add-service=squid
firewall-cmd --reload
```

------

## 六、在计算节点中使用 HTTP 代理

跳板机内网地址 192.168.0.100 ，Squid 默认端口 3128。

**临时添加代理：**

```
export http_proxy="http://192.168.0.100:3128"
export https_proxy="http://192.168.0.100:3128"
```

**永久添加代理：**

用 vi 打开 ~/.bashrc 文件，并添加上述两行配置。