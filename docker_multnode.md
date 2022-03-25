# docker配置多节点集群

## 1 镜像下载+配置

```shell
$ sudo docker pull centos:centos7
$ sudo run -it --name cen centos:centos7

# yum需要换源(centos8才需要)
$ cd /etc/yum.repos.d
$ sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
$ sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
$ yum update

$ yum install openssh-clients openssh-server openssl passwd net-tools vim which rsh rsh-server initscripts xinetd

$ exit
$ docker commit cen ccloud/centos
```

## 2 构建容器网络

```shell
$ sudo docker network create --subnet=172.18.0.0/16 mynetwork
$ sudo docker network ls
#master节点与宿主机建立端口映射，可以用scp进行文件传输
$ sudo docker run -d --privileged=true --name master -h master --add-host master:172.18.0.3 --add-host slave1:172.18.0.4 --add-host slave2:172.18.0.5 --net=mynetwork --ip=172.18.0.3 -p 8080:10001 ccloud/centos /usr/sbin/init

$ docker run -d --privileged=true --name slave1 -h slave1 --add-host master:172.18.0.3 --add-host slave1:172.18.0.4 --add-host slave2:172.18.0.5 --net=mynetwork --ip=172.18.0.4 ccloud/centos /usr/sbin/init

$ docker run -d --privileged=true --name slave2 -h slave2 --add-host master:172.18.0.3 --add-host slave1:172.18.0.4 --add-host slave2:172.18.0.5 --net=mynetwork --ip=172.18.0.5 ccloud/centos /usr/sbin/init
```

## 3 配置容器ssh免密

```shell
#master配置
$ sudo docker exec -it master /bin/bash

$ ssh-keygen -q -t rsa -b 2048 -f /etc/ssh/ssh_host_rsa_key -N ''
$ ssh-keygen -q -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key -N ''
$ ssh-keygen -t dsa -f /etc/ssh/ssh_host_ed25519_key -N ''
#生成RSA随机图,修改 /etc/ssh/sshd_config 配置信息：
#UsePAM yes 改为 UsePAM no 
#UsePrivilegeSeparation sandbox 改为 UsePrivilegeSeparation no
#具体执行如下：
$ sed -i "s/#UsePrivilegeSeparation.*/UsePrivilegeSeparation no/g" /etc/ssh/sshd_config
$ sed -i "s/UsePAM.*/UsePAM no/g" /etc/ssh/sshd_config
#启动sshd
$ /usr/sbin/sshd
#查看ssh服务是否启动成功
$ ps -ef | grep ssh
#修改root密码
$ passwd 
...
...

#生成密码对
$ ssh-keygen -t rsa
#查看生成的私钥idrsa和公钥idrsa.pub
$ cd ~/.ssh/
#查看ssh服务是否启动成功
$ ps -ef | grep ssh


```

```shell
#slave1配置
$ sudo docker exec -it slave1 /bin/bash

$ ssh-keygen -q -t rsa -b 2048 -f /etc/ssh/ssh_host_rsa_key -N ''
$ ssh-keygen -q -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key -N ''
$ ssh-keygen -t dsa -f /etc/ssh/ssh_host_ed25519_key -N '' 
$ sed -i "s/#UsePrivilegeSeparation.*/UsePrivilegeSeparation no/g" /etc/ssh/sshd_config
$ sed -i "s/UsePAM.*/UsePAM no/g" /etc/ssh/sshd_config
$ /usr/sbin/sshd
$ ps -ef | grep ssh
$ passwd 
$ ssh-keygen -t rsa
$ cd ~/.ssh/
$ ps -ef | grep ssh
$ /usr/sbin/sshd
#将slave1的公钥发送到master上
$ scp id_rsa.pub root@master:~/.ssh/id_rsa.pub.slave1

```

```shell
#slave2配置
$ sudo docker exec -it slave2 /bin/bash

$ ssh-keygen -q -t rsa -b 2048 -f /etc/ssh/ssh_host_rsa_key -N ''
$ ssh-keygen -q -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key -N ''
$ ssh-keygen -t dsa -f /etc/ssh/ssh_host_ed25519_key -N '' 
$ sed -i "s/#UsePrivilegeSeparation.*/UsePrivilegeSeparation no/g" /etc/ssh/sshd_config
$ sed -i "s/UsePAM.*/UsePAM no/g" /etc/ssh/sshd_config
$ /usr/sbin/sshd
$ ps -ef | grep ssh
$ passwd 
$ ssh-keygen -t rsa
$ cd ~/.ssh/
$ ps -ef | grep ssh
$ /usr/sbin/sshd
#将slave2的公钥发送到master上
$ scp id_rsa.pub root@master:~/.ssh/id_rsa.pub.slave2

```

```shell
#进入master的~/.ssh/,合并id_rsa.pub.*到authorized_keys中，然后再发送到slave1、slave2的~/.ssh/

$ cd ~/.ssh/
$ cat id_rsa.pub >> authorized_keys
$ cat id_rsa.pub.slave1 >> authorized_keys 
$ cat id_rsa.pub.slave2 >> authorized_keys
$ scp authorized_keys root@slave1:~/.ssh/
$ scp authorized_keys root@slave2:~/.ssh/
```

## 4 配置容器rsh免密

```shell
$ vi /etc/securetty
rexec
rlogin
rsh
$ vi /etc/xinetd.d/rlogin
# default: on
# description: rlogind is the server for the rlogin(1) program.  The server \
#       provides a remote login facility with authentication based on \
#       privileged port numbers from trusted hosts.
service login
{
        socket_type             = stream
        wait                    = no
        user                    = root
        log_on_success          += USERID
        log_on_failure          += USERID
        server                  = /usr/sbin/in.rlogind
        disable                 = no
}
$ vi /etc/xinetd.d/rsh
# default: on
# description: The rshd server is the server for the rcmd(3) routine and, \
#       consequently, for the rsh(1) program.  The server provides \
#       remote execution facilities with authentication based on \
#       privileged port numbers from trusted hosts.
service shell
{
        socket_type             = stream
        wait                    = no
        user                    = root
        log_on_success          += USERID
        log_on_failure          += USERID
        server                  = /usr/sbin/in.rshd
        disable                 = no
}

$ service xinetd restart

$ vi /etc/hosts.equiv
master
slave1
slave2
$ vi /root/.rhosts
master
slave1
slave2
```



## 5 本机和master传文件

```shell
#采用ssh,之前已经打通端口8080：10001
$ scp <route><filename> root@172.18.0.3:<route><filename>
$ scp root@172.18.0.3:<route><filename> <route><filename>

#采用docker cp
$ docker cp <route><filename> <container>:<route><filename>
$ docker cp <container>:<route><filename> <route><filename>
```

