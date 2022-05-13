

# 多节点配置HPL测试

## 1 安装MPICH

```shell
# 下载mpich的源码
$ wget --no-check-certificate https://www.mpich.org/static/downloads/3.3rc1/mpich-3.3rc1.tar.gz
$ tar zxvf mpich-3.3rc1.tar.gz #这里我用的是mpich-3.3
$ cd mpich-3.3rc1
$ ./configure -prefix=/usr/local/mpich #这个会运行一段时间
#其中prefix用来指定安装的路径，这样就可以将安装后的所有文件都放到同一个目录下了
$ make -j16 &&  make install #这条指令以及上一条指令是安装时常用的一套指令，会很久，慢慢等，j4是四线程
#  -j是几个线程编译，可以加快速度


# 配置环境变量
$ cd ~
$ vim .bashrc  #注意要到主目录中使用
$ export PATH=/usr/local/mpich/bin:$PATH
$ source .bashrc
```

## 2 安装GotoBLAS库

```shell
#在usr/local/mathlib/goto下解压：
$ tar -zxvf GotoBLAS2-1.13.tar.gz
$ cd GotoBLAS2
#centos7需要更改配置文件f_check
#最后部分更改为 
#print MAKEFILE "FEXTRALIB=$linker_L -lgfortran -lm -lquadmath -lm $linker_a\n";
$ make  CC=gcc BINARY=64 TARGET=NEHALEM -j16
```

## 3 配置HPL

```shell
#在用户目录下解压：
$ tar  –zxvf   hpl-2.1.tar.gz
$ cd hpl-2.1
#根据机器的情况复制Makefile模板：
```

![img](file:////tmp/wps-wxj/ksohtml/wpsi8GBVu.png)

```shell
$ cp  setup/Make.Linux_PII_CBLAS  Make.Linux
$ vi Make.Linux

#根据如下情况修改Makefile模板：
```



![4492A72CE101579FCBD75624D992DA1C](/home/wxj/.deepinwine/Deepin-QQ/dosdevices/c:/users/wxj/Application Data/Tencent/QQ/Temp/4492A72CE101579FCBD75624D992DA1C.jpg)![7E344DA49AFA3C6C01BE8B47EA991A05](/home/wxj/.deepinwine/Deepin-QQ/dosdevices/c:/users/wxj/Application Data/Tencent/QQ/Temp/7E344DA49AFA3C6C01BE8B47EA991A05.jpg)