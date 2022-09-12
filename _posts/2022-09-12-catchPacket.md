---
layout: post
title: "抓包软件使用：wireshark、tcpdunp"
date: 2022-09-12
tags: [kali, 渗透测试, wireshark, tcpdump]
comments: true
author: JumpWang


---

# wireshark

## 基本使用方法

​	选择网卡以开始，捕捉该网卡的网络包

​	假设选择eth0网卡，可以对该网卡的捕捉进行详细的设置：

- 混杂模式：开启的时候捕捉所有经由本网卡的数据包。不开启的时候只能捕捉到该网卡为终点的数据包，包括从该网卡发出以及收到的（包括以该网卡为目的地的单播以及组播和广播），假设我们的网卡是2，一个包从1发到3，中间途经2，那么不开启混杂模式的情况下是抓不到的。
- 捕获过滤器（capture filter）：可以使用一些命令指定要抓取哪些数据包，包括只抓特定ip，只抓tcp，只抓某端口等等（具体命令可以参考wireshark给的例子）
- 保存：尽量选择`.pcap`格式，因为几乎所有抓包软件都支持这个格式，而`pacpng`则是wireshark独有的格式
- 首选项：在外观可以增删列、改变布局和字体，在protocols里包含wireshark可以解码的所有协议类型及设置

## 过滤器

捕获过滤器：选择网卡开始抓包的时候设置的那个

显示过滤器：显示筛选器更常用、可以做更强的筛选，包括各种表达式的结合。

- eg：可以先搜索udp，然后选中一个包的ip，右键-> apply  as filter，然后就可以将当前的筛选与该ip进行各种逻辑结合，产生新的表达式。用这种方式可以逐层的将不想要的数据包剔除掉

## 常见协议包

​	这部分像ip、arp、tcp、udp在计网中详细的学过了，就不细说了

​	wireshark是通过端口号对协议进行解析的，比如HTTP是80端口，如果有的http包没走80端口，wireshark将无法识别出来，也就不会再解析tcp的上层协议， 这时候我们可以通过对该包进行右键， 选择decode as，手动设置为解析成http

## 数据流

​	在实际网络中通常是多个数据包甚至一片数据包合起来完成一件工作，一个个的看是非常麻烦的。这些数据包合起来构成了一个数据流，我们在可以在wireshark中右键follow tcp stream，来打开一个窗口观察整个数据流的交互。

​	常见数据流包括http、smtp、pop3、ssl等等

## 统计功能（static）

summary（捕获的文件属性）：可以查看该包的摘要信息

![image-20220713082741822](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20220713082741822.png)

endpoints（端点）：可以查看使用不同协议的节点，可以通过传输字节数进行排序（发送、接收)

![image-20220713082720806](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20220713082720806.png)

protocol hierarchy（协议分级）：可以了解到当前捕捉到的数据包的协议类型与所占比例，如果dns占比很大可能会有问题

![image-20220713082633219](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20220713082633219.png)

package length（分组长度）：可以看到当前捕获的文件是大包为主还是小包为主，小包很多会导致网络性能不好同时也可能正在遭受攻击

![image-20220713082558730](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20220713082558730.png)

conversation（会话）：可以看到不同地址之的会话，以及交换的信息量

![image-20220713082546071](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20220713082546071.png)

analysis->export info（专家信息）：好的专家系统可能会给我们带来很大的便利，专家信息会对数据包进行自动的分析

![image-20220713083534019](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20220713083534019.png)

## 缺陷和不足

​	当面对大的流量或分析大文件时，wireshark的性能上有所欠缺。基于这种不足，大型企业常常需要使用商业化的抓包产品，一般基于sniff和wireshark进行二次开发

# tcpdump

​	每个unix和linux系统默认安装，不支持GUI

- 抓包默认只抓68个字节，通常能抓到二级包头

- 开始抓包：

  `-i`指定抓包的网卡

  `-s`指定抓获的字节数，0表示有多少抓多少

  `-w`指定保存的文件

  ```sh
  tcpdummp -i eth0 -s 0 -w a.cap
  ```

  过滤器：

  ```sh
  tcpdump -i eth0 port 22
  ```

- 读包：

  显示概要

  ```sh
  tcpdump -r a.cap
  ```

  以ascii码的形式显示包里的内容：

  ```sh
  tcpdump -A -r a,cap
  ```

  显示过滤器：

  **通过操作系统的管道实现**

  `-n`表示不显示域名只使用ip地址

  `awk`在这里是只显示第三列的内容

  `sort -u`表示剔除重复项

  ```sh
  tcpdump -n -r http.cap | awk '{print $3}' | sort -u
  ```

  **通过tcpdump本身实现**

  只显示源/目标IP是1.1.1.1的

  ```sh
  tcpdump -n [src|dst] host 1.1.1.1 -r http.cap
  ```

  只显示53端口的

  ```sh
  tcpdump -n port 53 -r http.cap
  ```

  `-X`以16进制显示

  ```sh
  tcpdump -nX port 53 -r http.cap
  ```

  **高级过滤**

  flag字节为24的包（即psh+ack位置1）

  ````sh
  tcpdump -A -n 'tcp[13]=24' -r http.cap
  ````

# 