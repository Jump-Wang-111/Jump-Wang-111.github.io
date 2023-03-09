---
layout: post
title: "加密固件解包"
date: 2023-03-10
tags: [IoT, 固件安全]
comments: true
toc: true
author: JumpWang


---

​	对固件的分析可以简单的分为逆向工程以及具体分析技术。其中逆向工程可以简单划分为以下几个步骤：固件获取、固件解包、以及固件反编译反汇编。本篇文章只聚焦于固件解包这一过程，不讨论其他部分。

​	事实上，最近在跟着导师做固件方向的项目。在做项目的过程中我发现了一件事。我目前的固件逆向水准可以用一句话概括：上官网，下固件，`binwalk`一把梭。梭出来了，可以继续分析文件系统了；梭不出来......尝试换一个梭......

​	我寻思总这样也不太行，于是打算整理学习一下解包加密固件的方法。 

## 回溯未加密的旧固件

​	现在物联网发展方兴未艾，固件安全的保障机制也是在发展中一点点在完善的。针对我们讨论的加密也是这样，很可能加密在旧版本中还没有，新版本才加进去。或者旧版本的加密存在问题，需要更新加密的算法or机制。

​	而如果这个迭代的过程暴露在我们面前，这就是一个可利用的不安全因素。为什么这么讲？

​	根据[这篇博客](https://www.zerodayinitiative.com/blog/2020/2/6/mindshare-dealing-with-encrypted-router-firmware)，我们可以简单想象如下情形（图片引自该博客）：

1. 固件过去没有加密，某个版本之后开始加密。那么最后一个未加密的版本中必然包含解密的程序（不然他自己怎么更新？）。

   ![Figure 2: Firmware release scenario 1](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/Picture2.png)

2. 固件过去用方法1加密，现在用方法2加密，此时中间版本未加密。

   ![Figure 3: Firmware release scenario 2](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/Picture3.png)

3. 固件过去用方法1加密，现在用方法2加密，此时中间版本用方法1加密。当然，这种情况下就不存在可供我们使用的未加密旧固件了，想要获取解包后的固件需要寻找其他方法。

   ![Figure 4: Firmware release scenario 3](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/Picture4.png)

### D-Link DIR-882

#### 获取固件

​	以D-Link的DIR-882为例，这里我们在 [官网下载站](https://tsd.dlink.com.tw/ddetail)下载两个固件包，分别是最后一个未加密的过渡版本1.04B02版本（DIR882A1_FW104B02_Middle_FW_Unencrypt）和最新的1.30B06版本（DIR882A1_FW130B06）

​	其中1.04B02版本需要进入到v1.10版本的界面获取

![image-20221218155047223](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20221218155047223.png)

![image-20221218154636262](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20221218154636262.png)

​	其中过渡版本并未加密，可以binwalk解包，而最新的版本是加密的

![image-20221218155014212](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20221218155014212.png)

#### 定位解密文件

​	这里我们效仿[OneShell师傅的做法](https://paper.seebug.org/1651/)，对文件系统进行字符串搜索，期望查找到解密的关键字

​	由于固件一般支持在线更新。所以我们可以解包后按如下逻辑搜索，定位固件在线升级的网页->检索相关的后端文件（包括http服务器、js、以及cgi等文件）-> 获取相关字符串 -> 找到解密文件

​	定位升级的网页，从这里可以看到对应后端是lighttpd

```sh
find . -iname "*htm*" | grep -i firm
```

![image-20221218170714865](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20221218170714865.png)

​	从js中可以看到与两个文件有关

```sh
find . -iname "*js*" | grep -i update
find . -iname "*js*" | grep -i firm
```

![image-20221218171024666](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20221218171024666.png)

​	而从文件的具体内容来看，很遗憾的是js脚本的工作只有验证和下载固件的部分，之后的对更新固件解密的操作并不在这里

```sh
find . -iname "*js*" | grep -i firm | xargs grep -ih firm
```

![image-20221218171138177](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20221218171138177.png)

​	搜索http服务器，~~好的啥都没搜到~~

```sh
find . -iname "*httpd*" | xargs strings | grep -i firm
```

![image-20221218171214026](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20221218171214026.png)

​	搜索cgi，终于找到了想要的东西（OneShell师傅的文章里的DIR3040也是在这里找到的，由此可见同一厂商内的不同产品彼此间有很大参考价值），解密的应该是二进制文件，在`/bin/imgdecrypt`

```sh
find . -iname "*cgi*" | xargs strings | grep -i firm
```

![image-20221218171254293](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20221218171254293.png)

#### 解密

​	如果我们只想对加密固件解包，那么到这里找到解密程序就已经可以实现了。

​	使用file查看解密程序的架构，可以发现是32位的mips小端，弄清楚这个之后，我们就可以使用qemu模拟对应环境执行该程序了

![image-20221218191241447](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20221218191241447.png)

​	这里我们使用`qemu-mipsel-static`程序来进行模拟，其中mips指架构类型，el指小端，static表示该程序是静态编译，因此不需要动态链接库的支持，使用起来比较方便

```sh
# 将qemu-mipsel-static复制到当前目录下
cp $(which qemu-mipsel-static) .
# 使用chroot将当前目录作为新的root目录，并执行./qemu-mipsel-static，向其传递参数/bin/sh
sudo chroot . ./qemu-mipsel-static /bin/sh
```

​	如图所示，我们成功用模拟器跑起来了该路由器的shell

![image-20221218191949628](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20221218191949628.png)

​	接下来提前将要解压的加密固件放到固件文件系统的/tmp下，就可以按照我们之前看到的，将/tmp/firmware.img作为参数传递给解密程序，运行得到`.firmware.orig`，这里注意所有相关的路径尽量不要发生变化，因为可能会影响到解密程序的执行，因为不知道`imgdecryt`是否引用了其他文件。~~另外，我这里一时着急，忘了将加密固件更名为firmware.img，不过也跑成了，问题不大~~

![image-20221218191138153](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20221218191138153.png)

​	可以看到，执行过解密后的固件binwalk已经可以解析了（这里在执行过程中看到产生了`.firmware.orig`中间文件，执行后中间文件被删除，然后覆写了DIR882A1_FW130B06.bin）

![image-20221218193905221](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20221218193905221.png)

## 分析已存在的解密脚本

​	我们可以尝试在公网上搜索目标固件的解密脚本，尤其是github等开源社区，或许已经有前人做过研究。如果能够搜索到，问题自然迎刃而解；如果不能，也可以尝试搜索同产品其他型号的现存解密脚本，或许可以直接使用，即使不能，通过分析解密脚本，可以获得其他型号加密固件的加密结构，对解密当前固件可能会有一定启发。

### D-Link DIR-882/3060

​	[该github仓库](https://github.com/0xricksanchez/dlink-decrypt)中包含了对DIR-882和DIR-3060适用的解密脚本，作者从买来的路由器中提取了`imgdecrypt`可执行文件，并适用IDA对其进行逆向，还原了解密的步骤。同时，只要逆向了main函数就能发现，该可执行文件中不仅包含解密函数，同时包含加密函数，图片如下所示（引自[Breaking the D-Link DIR3060 Firmware Encryption](https://0x00sec.org/t/breaking-the-d-link-dir3060-firmware-encryption-recon-part-1/21943)），在原文章中对逆向过程做了非常精彩的讲解，我这里就不画蛇添足了。

![img](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/5d2e11cea43a20b189bb4a51fb84c8f2eca397c7.png)

​	这里直接贴出对解密脚本分析得到的固件包结构：

![struct.drawio](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/struct.drawio.png)



## 分析“兄弟”固件

​	从[加密固件之依据老固件进行解密](https://paper.seebug.org/1651/)中可以看到，DIR-3040与DIR-882具有相同的结构，由此可见“兄弟”固件之间是可能具有类似甚至相同的结构的，值得参考。

## 从硬件设备中提取

​	~~过于硬核，放弃尝试~~

## 参考链接

[MindShaRE: Dealing with encrypted router firmware](https://www.zerodayinitiative.com/blog/2020/2/6/mindshare-dealing-with-encrypted-router-firmware)

[D-Link 路由器固件解密](https://yjy123123.github.io/2021/05/28/D-Link-%E8%B7%AF%E7%94%B1%E5%99%A8%E5%9B%BA%E4%BB%B6%E8%A7%A3%E5%AF%86/)

[加密固件之依据老固件进行解密](https://paper.seebug.org/1651/)

[Breaking the D-Link DIR3060 Firmware Encryption](https://0x00sec.org/t/breaking-the-d-link-dir3060-firmware-encryption-recon-part-1/21943)
