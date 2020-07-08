---
title: "深度学习-CNN训练Flabby bird  突破一万分"
date: 2017-1-23 19:06
tags:
- DL
- DEMO
categories:
- Technology
---

![](/image/bird/bird.gif)

环境 opencv/mxnet/python

1.安装opencv (ps:编译mxnet首先需要cv库)

``` bash
$ yum install cmake gcc gcc-c++
```

<!-- more -->
使用yum源安装以上工具，源配置可以设置为ali的yum源（ps：国内访问很快）

安装后测试cmake --version测试版本看是否成功

然后down opencv源码

``` bash
$ wget https://github.com/Itseez/opencv/archive/2.4.13.zip
``` 

down到安装目录，然后解压unzip 2.4.13.zip
进入opencv目录后
检查是否有CMakeLists.txt文件 该文件是用来生成makefile的
然后：

``` bash
$ make
$ make install
``` 

安装opencv
等待安装ing。。。。最好配置( -j8)不然比较慢,安装完成后 需要配置cv的lib*.so文件
主要有以下3种方式：（1）用ln将需要的so文件链接到/usr/lib或者/lib这两个默认的目录下边 

``` bash
$ ln -s /usr/local/lib/*.so /usr/lib
$ sudo ldconfig
``` 

（2）修改LD_LIBRARY_PATH

``` bash
$ export LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH
$ sudo ldconfig
``` 

（3）修改/etc/ld.so.conf vim /etc/ld.so.conf 
加一行

``` bash
$ /usr/local/lib
$ sudo  ldconfig
```

然后进入samples目录测试opencv安装是否成功

``` bash
$ cd /samples/c
$ ./facedetect --cascade="/usr/local/share/OpenCV/haarcascades/haarcascade_frontalface_alt.xml" --scale=1.5 lena.jpg
``` 

如果出现
![](/image/bird/1.jpg)
则表示opencv安装完成
接着需要配置PKG_CONFIG_PATH路径

``` bash
$ vi  /etc/bashrc
``` 

末尾加入一句

``` bash
$ export PKG_CONFIG_PATH=/usr/local/lib/pkgpath:$PKG_CONFIG_PATH
$ source /etc/bashrc
``` 

2.安装mxnet平台（ps：进入正题）
 mxnet 是一个非常优秀的深度学习的框架。足够灵活，速度足够快，扩展新的功能比较容易。
 
``` bash
$ git clone --recursive https://github.com/dmlc/mxnet
``` 

这里提醒注意一定不要忘记--recursive参数，因为mxnet依赖于DMLC通用工具包http://dmlc.ml/，--recursive参数可以自动加载mshadow等依赖
编译mxnet平台需要支持c++11 gcc4.8.x版本以上才支持c++11
所以需要升级gcc版本 yum源中是没有4.8以上版本的所以需要手动升级
gcc4.x版本保留用来编译gcc4.8x可以参考此处升级：
http://www.mudbest.com/centos%E5%8D%87%E7%BA%A7gcc4-4-7%E5%8D%87%E7%BA%A7gcc4-8%E6%89%8B%E8%AE%B0/
编译gcc的过程比较耗时 最好加上make -j4 参数
如果遇到类似*** [all-stage1-gcc] Error 2的bug 可以修改下 系统的swappiness虚拟交换内存解决
工具准备好后
下面进入正题编译mxnet
mxnet用到了大量线性代数运算，需要用到blas库
如果没安装openBlas，可以通过以下方式安装

``` bash
$ yum -y install blas blas-devel lapack lapack-devel atlas atlas-devel –nogpgcheck
``` 

并且在make目录下修改config.mk中
USE_BLAS = openblas
单纯使用CPU进行训练的过程是非常慢的可以mxnet可以通过CUDA调用GPU的处理能力
提升训练速度，通过配置config.mk文件配置是否编译CUDA，cuda安装可以到
http://developer.download.nvidia.com/compute/cuda/repos下载对应版本
安装过程参考此处
http://blog.csdn.net/a350203223/article/details/50262535
cuda安装完成后，在config.mk中配置如下：
USE_CUDA = 1
USE_CUDA_PATH = cuda安装路径
之后编译mxnet目录：

``` bash
$ make -j4
``` 

本文深度学习使用python平台：需要用到python环境

3.安装python环境

``` bash
$ yum -y install python pip-python
```
 
安装python和pip pip是python插件安装工具相当于centos的yum
然后安装pygame包，flabby bird的程序运行用到了pygame模块：

``` bash
$ pip install pygame
$ cd mxnet/python
$ python setup.py install
``` 

安装mxnet模块 现在的版本是0.7.0
等待安装完成 马上就能跑了。。
安装完成后进入/mxnet/example/image-classification目录

``` bash
$ python train_mnist.py
``` 

![](/image/bird/2.jpg)
表示训练完成，mxnet模块安装完成
接着需要我们要去下载flabby bird python源码

``` bash
$ git clone https://github.com/learn2Pro/DRL-FlappyBird.git
``` 

下载完成后 我们还需要python的opencv模块
然后将/lib/python-version/cv2.so库文件放到python安装目录下
然后进入DRL-FlappyBird目录

``` bash
$ python FlappyBirdDQN.py
``` 

最后效果图 ，接下来就是等待了，如果只用cpu训练是非常慢的，可以调用cuda 使用GPU加速，GPU的并行运算能力可以提升20-30倍

![](/image/bird/bird.gif)



