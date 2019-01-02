---
layout: post
title:  【指南向】ubuntu18.04装机tensorflow-gpu
date: 2019-01-03 00:13:24 +0800
comments: false
---

### 背景

找计算资源跑CNN demo. rmbp没有gpu，跑一个mini batch都要好几秒（单次trainning CPU狂转估计得跑十来分钟）。阿里云/腾讯云的GPU实例贵得吓人，有一个极客云很便宜，但是感觉很不正规。调查下来，Google Cloud注册送钱，生成一个最低配的实例，这些钱可以用很久。于是就他了。

### 我选的安装栈

- ubuntu18
- nvidia driver
- cuda toolkits（主要是cuda9.0, cudnn）
- miniconda
- tensorflow-gpu

### 废话
选ubuntu是因为据说跑DL用得最多，出问题好找，选择了最新的ubuntu。 nvidia driver与cuda套件据说比较经常在nvidia官网找版本手动安装，但是我嫌麻烦，用的是ubuntu-drivers工具。装miniconda是为了方便装tensorflow-gpu系列。没有选择anaconda是因为整个套件太大了（12G）...磁盘占用是要钱的。总之，就是在拒绝任何手工编译的原则下，完成安装。tensorflow官网有指导ubuntu16安装gpu套件，因为发行版不匹配我没有尝试...

### 来了

首先在google cloud平台上生成一个VM实例，我选的是单核CPU+2G内存，（因为计划跑python...多核应该用处不大），GPU不是每个机房都有，选台湾，1个GPU+12G内存，Nvidia Tesla K80，基本就是最便宜的套路了。系统选ubuntu+20G SSD（磁盘选太小了其实）。（后来发现VM Instance关机的时候是不收钱的...其实可以创建一个配置更高的实例，不用的时候关机就好了）

这里有个麻烦的事情是，google cloud在IAM管理-配额里面可以看到GPU数量限制是0，需要在他的界面上请求加数量，（我加的是1），据说加多了会要求充值，不让白嫖...这个审核是人工的，可能得要一点时间，我差不多等了一个小时不到。

管理这个VM，从网页命令行也可以，我使用了gcloud在我本机的命令行远程操作，需要看一点谷歌的说明配置命令。

#### 安装显卡驱动

参考`https://linuxconfig.org/how-to-install-the-nvidia-drivers-on-ubuntu-18-04-bionic-beaver-linux`安装N卡驱动，我这里的版本是nvidia-390.77，然后重启。

主要的命令是

```shell
$ ubuntu-drivers devices
```

```shell
$ sudo ubuntu-drivers autoinstall
```

完事重启以后，

```shell
$ nvidia-smi
```

查看是否成功

#### 运行nvidia-docker检查一下驱动

这里我选择先跑跑看docker验证一下驱动是不是装好了...免得再然后报错不知道到底是驱动的问题还是cuda套件的问题


顺着docker官方指南，安装docker-ce

`https://docs.docker.com/install/linux/docker-ce/ubuntu/#set-up-the-repository
`

安装完docker-ce，按nvidia-docker readme安装支持

`https://github.com/NVIDIA/nvidia-docker`

跑完readme的安装与验证不算，我还跑了一个tensorflow官网上的docker实例

`https://www.tensorflow.org/install/docker`

```shell
$ docker run -it --rm tensorflow/tensorflow \
   python -c "import tensorflow as tf; tf.enable_eager_execution(); print(tf.reduce_sum(tf.random_normal([1000, 1000])))"
```

完美运行...心满意足去装cuda

#### 安装tensorflow-gpu运行环境

主要需要安装的是cuda和cuDNN，cuDNN就是专门为深度神经网络加速的一个组件。安装了conda之后，这两个东西都可以傻瓜安装，无需编译。

安装conda

`https://conda.io/miniconda.html`

执行官方的安装脚本，顺着他安装。

完事之后，创建一个conda env
```shell
$ conda create -n test python=3.6 
$ conda activate test
```
然后参考`https://mc.ai/install-tensorflow-gpu-to-use-nvidia-gpu-on-ubuntu-18-04-do-ai/`
最后部分，使用conda安装所有的python开发套件

```shell
$ conda install \
 tensorflow-gpu==1.10.0 \
 cudatoolkit==9.0 \
 cudnn=7.1.2 \
 h5py
```

按照文章中的简单python脚本，测试tensorflow在gpu上的运行

```python
import tensorflow as tf
print("tf version = ", tf.__version__)
with tf.device('/gpu:0'):
    a = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[2, 3], name='a')
    b = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[3, 2], name='b')
    c = tf.matmul(a, b)

with tf.Session() as sess:
    print (sess.run(c))
```
再来一个tensorflow官方给的gpu demo

`https://www.tensorflow.org/guide/using_gpu`

```python
# Creates a graph.
with tf.device('/device:GPU:0'):
  a = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[2, 3], name='a')
  b = tf.constant([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], shape=[3, 2], name='b')
  c = tf.matmul(a, b)
# Creates a session with log_device_placement set to True.
sess = tf.Session(config=tf.ConfigProto(log_device_placement=True))
# Runs the op.
print(sess.run(c))
```

#### 跑一个正式的demo

运行`https://github.com/astorfi/TensorFlow-World/tree/master/codes/3-neural_networks/convolutional-neural-network`
中CNN在MINIST数据集上的例子（也就是我在本机CPU版本上十几分钟都跑不完的demo）

注意，在`train.sh`中修改`--log_device_placement`置为True，观察gpu是否被使用了。

结果十几秒就跑完了一次trainning...相比之下，我RBMP上的3.1GHz的CPU也算是苹果笔记本里能打的了...跑浮点GPU厉害太多了...

### 总结

下次磁盘可以开大点...CPU，内存，GPU都是可以事后加的，磁盘分区不能直接扩容，挺麻烦的。

以及

白嫖，爽~