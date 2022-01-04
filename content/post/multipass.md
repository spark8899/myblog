---
title: "Multipass"
date: 2022-01-04T17:15:24+08:00
draft: true
---

# install multipass
```shell
$ brew install --cask multipass
multipass version

# 查找镜像
$ multipass find
Image                       Aliases           Version          Description
18.04                       bionic            20211129         Ubuntu 18.04 LTS
20.04                       focal,lts         20211129         Ubuntu 20.04 LTS
21.04                       hirsute           20211130         Ubuntu 21.04
21.10                       impish            20211103         Ubuntu 21.10
anbox-cloud-appliance                         latest           Anbox Cloud Appliance
minikube                                      latest           minikube is local Kubernetes

# 安装
$ multipass launch ubuntu

# 检查运行实例
$ multipass list

# 连接运行实例
$ multipass shell dancing-chipmunk

# 连接实例运行命令
$ multipass exec dancing-chipmunk -- lsb_release -a

# 停止一个实例
$ multipass stop dancing-chipmunk

# 删除一个实例
$ multipass delete dancing-chipmunk

# 清理文件
$ multipass purge

# 帮助文件
$ multipass --help
```

# install podman
```
### start run virtual machine
$ multipass launch -c 2 -d 10G -m 2G -n podman
 * -n : 指定启动实例名字
 * -c : 分配 CPU 数量
 * -d : 设置磁盘容量
 * -m : 设置内存容量

# multipass list
# multipass shell podman

### start run podman
ubuntu@podman:~$ . /etc/os-release
ubuntu@podman:~$ echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/ /" | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
ubuntu@podman:~$ curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/Release.key | sudo apt-key add -
ubuntu@podman:~$ sudo apt update
ubuntu@podman:~$ sudo apt -y upgrade
ubuntu@podman:~$ sudo apt -y install podman

### create podman socket
ubuntu@podman:~$ sudo systemctl cat podman.socket
# /lib/systemd/system/podman.socket
[Unit]
Description=Podman API Socket
Documentation=man:podman-system-service(1)

[Socket]
ListenStream=%t/podman/podman.sock
SocketMode=0660

[Install]
WantedBy=sockets.target

ubuntu@podman:~$ sudo systemctl cat podman.service
# /lib/systemd/system/podman.service
[Unit]
Description=Podman API Service
Requires=podman.socket
After=podman.socket
Documentation=man:podman-system-service(1)
StartLimitIntervalSec=0

[Service]
Type=notify
KillMode=process
ExecStart=/usr/bin/podman system service

ubuntu@podman:~$ sudo systemctl enable podman.socket --now
ubuntu@podman:~$ podman --remote info

### install podman cli
$ brew install podman
add localhost ~/.ssh/id_rsa.pub to podman's /root/.ssh/authorized_keys
$ podman system connection add ubuntu --identity ~/.ssh/id_rsa ssh://root@192.168.64.2/run/podman/podman.sock
$ podman system connection list
$ podman ps
$ podman pull nginx:alpine
$ podman image ls
podman system connection default <NAME>


### run mongodb
# multipass shell podman
ubuntu@podman:~$ sudo mkdir /var/lib/mongodb
ubuntu@podman:~$ sudo podman run -d --name mongod -p 27017:27017 -v /var/lib/mongodb:/data/db:Z mongo
```
