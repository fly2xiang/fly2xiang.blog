title: "Linux screen 命令"
date: 2014-12-01 22:14:35
tags:
- Linux
- 服务器
- 命令
- Screen
categories: 
- 服务器

---

## 简介

在Linux服务器上需要执行长时间的命令，比如网络传输时，是不能断开连接的，否则进程会被杀掉。screen是一个命令行终端管理软件，使用它可以在关闭远程链接的情况下继续运行命令而不中断。

## 常用命令

`screen` 新建一个窗口，可以在这个窗口中执行任何命令

`screen -ls` 查看所有打开的窗口

`screen -r` 重新连接窗口

`screen -x` 重新连接上一个窗口

快捷键：`Ctrl+A`

`d` 暂时离开当前窗口

向另外一个服务器传输文件的例子：

登录到服务器上：

```
C:\ssh root@server1
```

在server1上执行screen，若服务器上没有安装screen，执行以下安装（CentOS）。

```sh
yum install screen
```

在新建的窗口中向另外的服务器传输文件

```
scp 200mfile.bin root@server2:
```

这时，传输正在进行，可以直接关掉ssh窗口，或者按`Ctrl+A` 再按 `d`暂时离开当前窗口。

10分钟后再次连接到服务器上，执行 `screen -ls` 看到所有打开的窗口

```
screen -ls
```

执行 screen -r PID 或者直接执行screen -x 重新连接传输文件的窗口，看到文件已经传输完成。

```
screen -x
```

## 共享窗口

你可以在服务器上新建一个窗口后告诉你朋友，他的终端中同时连接到同一个窗口，这样，你们可以共享同一个窗口，可以同步演示操作了。