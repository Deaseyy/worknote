# 配置免密远程登录

因为是要远程登录，那么需要通过使用ssh进行密钥对登录，这样每次登录服务器就可以不用输入密码了。

先来一句官方介绍：

> ssh 公钥认证是一种方便、高安全性的身份验证方法，它将本地“私有”密钥与远程主机上与用户关联的“公共”密钥进行匹配，从而实现免密登录。

**下面以本地windos主机连接远程linux服务器为例**，mac及linux用户可以查看 [官方文档](https://link.zhihu.com/?target=https%3A//code.visualstudio.com/docs/remote/troubleshooting%23_configuring-key-based-authentication)。



### 1.生成ssh密钥对（公私钥）

1、**首先检查本地是否有已生成ssh密钥对**

Linux用户查看是否存在公钥文件 `~/.ssh/id_rsa.pub`和私钥文件`~/.ssh/id_rsa`。windows用户查看：`C:\Users\12395\.ssh`。

2、**如果没有，则用如下命令生成，一路回车即可：**

```text
ssh-keygen -t rsa -b 4096
```



### 2、将生成的公钥添加到远程主机。

**将本地公钥文件`id_rsa.pub` 的内容添加（追加）到远程主机用户目录下 `.ssh` 文件夹内名为 `authorized_keys` 的文件中（可能已有其他公钥，追加到下面即可）。**

如果本地是linux主机，不用去复制粘贴，使用命令`ssh-copy-id`来完成，

```
ssh-copy-id 远程主机用户名@远程主机ip
```

输出结果如下：

```text
~$ ssh-copy-id remote_user@remote_id
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
remote_user@remote_id's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'remote_user@remote_id'"
and check to make sure that only the key(s) you wanted were added.
```



> **当往 `authorized_keys` 中添加公钥内容时，可能出现Permission denied**

​		这是因为远程服务器的authorized_keys文件不允许被修改。一般很自然会想到去查看修改文件的读写权限，执行`chmod 777 authorized_keys` 后发现又会报错权限不允许，错误如下：`chmod: changing permissions of ‘authorized_keys’: Operation not permitted`。

执行下面的命令查看当前文件属性，可以发现有 i 和 a 两个属性.

```
lsattr authorized_keys
```

原因：**在linux系统下，有些配置文件是不允许被任何人（包括root）修改的，为了防止被误修改或删除，可以设定该文件的不可修改位：immutable。**

> 说明此时的文件是被锁定的，任何用户都是修改不了的，那么我们就去掉这两个属性：

```
chattr -ia authorized_keys 
```

说明，减号（-）代表去掉的意思，反之加号（+）代表增加的意思

```
防止关键文件被修改：
　　chattr +i authorized_keys
如果需要修改文件则：
　　chattr -i authorized_keys
```

> 此时，文件的rwx权限就能被修改了

```
chmod 777 authorized_keys
```

> 我们再将公钥文件内容添加到 `authorized_keys`中，就能成功了。

- linux可通过命令 ssh-copy-id -i
- 或者直接复制内容，粘贴进去



### 3.远程连接测试

执行ssh命令：

```
ssh "用户名@服务器ip"
```

输出如下：

```
C:\Windows\System32>ssh "root@120.77.206.76"
Last login: Sun Jul  4 10:51:43 2021 from 223.73.210.73

Welcome to Alibaba Cloud Elastic Compute Service !

[root@izwz9el5vpkcpm2b97lzpqz ~]#
```

可以看到，我们无需输入密码，已经连接到远程服务器了。



# VScode 远程开发配置

参考： https://zhuanlan.zhihu.com/p/95678121

其他参考：https://blog.csdn.net/qq_29935433/article/details/105602026



打开项目时，弹出问题：

Could not establish connection to "my_server": Downloading VS Code Server failed - please install either curl or wget on the remote.

翻译：无法建立连接到"my_server":下载VS Code Server失败-请在远程上安装curl和wget。

远程需要去下载`wget` 和 `curl` 工具：执行 `yum -y install wget`，`yum -y install curl`。