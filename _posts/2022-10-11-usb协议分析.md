---
layout: post
title: "从wireshark抓包进行USB协议分析"
date: 2022-10-11
tags: [wireshark, usb, usbhid]
comments: true
toc: true
author: JumpWang

---

​	本文将在windows下从wireshark抓包的角度对USB协议进行分析

## USB

​	在抓包的过程中我的电脑上一共连接了3个usb设备，分别为一个键盘、一个无线鼠标接收器以及一个U盘

​	下面是我的设备管理器的显示，由于电脑上除了我外连的三个usb设备之外还有电脑内的主机控制器和根集线器等设备，所以设备管理器显示有6个。

![image-20220726174259218](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20220726174259218.png)

​	usb设备是三段地址描述，**第一个是总线，第二个是设备地址，第三个是端口.**我们可以使用这种方式在wireshark内进行过滤

![image-20220904135532735](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20220904135532735.png)

​	我们可以看到图中有一部分数据为`USB URB`，那么什么是URB呢？根据microsoft的官方文档我们可以看到URB的定义。[相关网页](https://docs.microsoft.com/zh-cn/windows-hardware/drivers/usbcon/communicating-with-a-usb-device)

> ​	通用串行总线 (USB) 客户端驱动程序无法直接与其设备通信。 相反，客户端驱动程序创建请求并将其提交到 USB 驱动程序堆栈进行处理。 在每个请求中，客户端驱动程序提供长度可变的数据结构，称为 *USB 请求块 (URB)* 。 [**URB**](https://docs.microsoft.com/zh-cn/windows-hardware/drivers/ddi/usb/ns-usb-_urb)结构描述请求的详细信息，还包含有关已完成请求的状态的信息。 客户端驱动程序通过 URBs 执行所有设备特定的操作，包括数据传输。 在将 URB 提交到 USB  驱动程序堆栈之前，客户端驱动程序必须用该请求的相关信息对其进行初始化。 对于某些类型的请求，Microsoft 提供了 helper  例程和宏，用于分配 **URB** 结构并使用客户端驱动程序提供的详细信息填充 **URB** 结构的必要成员。

### 接入过程分析

#### 获取设备描述符

​	我们以插入的键盘（地址为7）为例进行分析

​	可以看到，主机首先向1.7.0发送了一个GetDescriptor请求，这里请求的是设备描述符。

![image-20220904140744414](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20220904140744414.png)

​	之后我的键盘向主机返回了一个18字节的设备描述符，里面指明了usb版本为2.0，产品id为阿米洛的键盘，可能的配置数为1等等信息。

![image-20220904141010068](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20220904141010068.png)

​	设备描述符结构如下所示：

```c
struct _DEVICE_DESCRIPTOR_STRUCT 
{ 
    BYTE bLength;           //设备描述符的字节数大小，为0x12 
    BYTE bDescriptorType;   //描述符类型编号，为0x01 
    WORD bcdUSB;            //USB版本号 
    BYTE bDeviceClass;      //USB分配的设备类代码，0x01~0xfe为标准设备类，0xff为厂商自定义类型 
   						  //0x00不是在设备描述符中定义的，如HID 
    BYTE bDeviceSubClass;   //usb分配的子类代码，同上，值由USB规定和分配的 
    BYTE bDeviceProtocol;   //USB分配的设备协议代码，同上 
    BYTE bMaxPacketSize0;   //端点0的最大包的大小 
    WORD idVendor;          //厂商编号 
    WORD idProduct;         //产品编号 
    WORD bcdDevice;         //设备出厂编号 
    BYTE iManufacturer;     //描述厂商字符串的索引 
    BYTE iProduct;          //描述产品字符串的索引 
    BYTE iSerialNumber;     //描述设备序列号字符串的索引 
    BYTE bNumConfiguration; //可能的配置数量 
}
```

> - bLength : 描述符大小．固定为0x12．
>
> - bDescriptorType : 设备描述符类型．固定为0x01．
>
> - bcdUSB : USB 规范发布号．表示了本设备能适用于那种协议，如2.0=0200，1.1=0110等．
>
> - bDeviceClass : 类型代码（由USB指定）。当它的值是0时，表示所有接口在[配置描述符](https://www.usbzh.com/article/detail-67.html)里，并且所有接口是独立的。当它的值是1到FEH时，表示不同的接口关联的。当它的值是FFH时，它是厂商自己定义的．
>
> - bDeviceSubClass : 子类型代码（由USB分配）．如果bDeviceClass值是0，一定要设置为0．其它情况就跟据USB-IF组织定义的编码．
>
> - bDeviceProtocol : 协议代码（由USB分配）．如果使用USB-IF组织定义的协议，就需要设置这里的值，否则直接设置为0。如果厂商自己定义的可以设置为FFH．
>
>   > 操作系统使用bDeviceClass、bDeviceSubClass和bDeviceProtocol来查找设备的类驱动程序。通常只有  bDeviceClass 设置在设备级别。大多数类规范选择在接口级别标识自己，因此将 bDeviceClass 设置为  0x00。这允许一个设备支持多个类，即USB复合设备。
>
> - bMaxPacketSize0 : 端点０最大分组大小（只有8,16,32,64有效）．
>
> - [idVendor](https://www.usbzh.com/article/detail-953.html) : 供应商ID（由USB分配）．
>
> - idProduct : 产品ID（由厂商分配）．由供应商ID和产品ID，就可以让操作系统加载不同的驱动程序．
>
> - bcdDevice : 设备出产编码．由厂家自行设置．
>
> - iManufacturer : 厂商描述符字符串索引．索引到对应的[字符串描述符](https://www.usbzh.com/article/detail-53.html)． 为０则表示没有．
>
> - iProduct : :产品描述符字符串索引．同上．
>
> - iSerialNumber : 设备序列号字符串索引．同上．
>
> - bNumConfigurations : 可能的配置数．定义设备以当前速度支持的配置数量

#### 获取配置描述符

​	这之后，主机又向1.7.0发出了获取描述符的请求，这次要获取的是配置描述符

![image-20220904141709181](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20220904141709181.png)

​	配置描述符定义了设备的配置信息，一个设备可以有多个配置描述符，大部分的USB设备只有一个配置描这符

​	读取配置描述符时，它会返回整个配置层次结构，其中包括所有相关的接口和[端点描述符](https://www.usbzh.com/article/detail-56.html)。wTotalLength字段反映配置描述符层次结构中的字节数

​	配置描述符在USB设备的枚举过程中，需要获取两次：第一次只获取配置描这符的基本长度9字节，获取后从wTotalLength字节中解析出配置描述符的总长度，然后再次获取全部的描述符

​	配置描述符结构如下：

```c
struct _CONFIGURATION_DESCRIPTOR_STRUCT 
{ 
    BYTE bLength;           //配置描述符的字节数大小，固定为0x09
    BYTE bDescriptorType;   //描述符类型编号，为0x02 
    WORD wTotalLength;     //返回整个数据的长度．指此配置返回的配置描述符，接口描述符以及端点描述符的全部大小 
    BYTE bNumInterface;     //此配置所支持的接口数量 
    BYTE bConfigurationVale;   //Set_Configuration命令需要的参数值 
    BYTE iConfiguration;       //描述该配置的字符串的索引值 
    BYTE bmAttribute;           //供电模式的选择，Bit4-0保留，D7:总线供电，D6:自供电，D5:远程唤醒
    BYTE MaxPower;             //设备从总线提取的最大电流 
}CONFIGURATION_DESCRIPTOR_STRUCT
```

> - bLength : 描述符大小．固定为0x09．
> - bDescriptorType : 配置描述符类型．固定为0x02．
> - wTotalLength : 返回整个数据的长度．指此配置返回的配置描述符，[接口描述符](https://www.usbzh.com/article/detail-64.html)以及[端点描述符](https://www.usbzh.com/article/detail-56.html)的全部大小．
> - bNumInterfaces : 配置所支持的接口数．指该配置配备的接口数量，也表示该配置下接口描述符数量．
> - bConfigurationValue : 作为Set Configuration的一个参数选择配置值．
> - iConfiguration : 用于描述该配置[字符串描述符](https://www.usbzh.com/article/detail-53.html)的索引．
> - bmAttributes : 供电模式选择．Bit4-0保留，D7:总线供电，D6:自供电，D5:远程唤醒．
> - MaxPower : 总线供电的USB设备的最大消耗电流．以2mA为单位．
> - 接口描述符：接口描述符说明了接口所提供的配置，一个配置所拥有的接口数量通过配置描述符的bNumInterfaces决定。

​	可以看到，1.7.0先是向主机返回了9个字节的配置描述符，并在其中指明了总长度为91

![image-20220904142921055](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20220904142921055.png)

​	而后主机又向设备发送了对配置描述符的请求，而这次1.7.0返回了91字节的全部配置描述符，里面包括了3个接口描述符，4个端点描述符和3个HID描述符，HID描述符附属的描述符的类型都是HID Report，4个端点为0-3，也就是说我们还有1.7.1-1.7.3可以进行交互，这里就不依次展开截图了

![image-20220904143119473](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20220904143119473.png)

![image-20220904143207537](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20220904143207537.png)

​	后面还有一些无法解析的包以及三次获取字符串描述符：获取阿米洛自己的名字字符串的包，这里不详述

![image-20220904154834568](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20220904154834568.png)

## USBHID

​	因为我们的设备是键盘，是一种HID设备，所以在完成接入的准备后会有USBHID协议的部分，这里我们继续做分析

​	首先主机向1.7.0发送了SET_IDLE请求，SET_IDLE请求会使HID设备相关的中断管道（端点）停止定时上报报告数据，直到有新的事件（有效数据）或直到的SET_IDLE时才继续上报报告数据。

​	HID设备以中断的方向进行上报数据给方机，比如说USB鼠标键盘，当无操作时，设备无须上报给数据给主机。不过USB设备的中断其实是轮询方式的，也就是说无论你是不是上报数据，主机都会发送IN的请求事务，这样会造成USB总线带宽的浪费。

![image-20220904155117093](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20220904155117093.png)

​	这之后则是请求HID报表描述符了，具体可以看[HID报表描述符原理解释](https://www.usbzh.com/article/detail-877.html)，这里就不贴出来了，HID报表描述符主要用来描述符USB HID设备上报的数据信息格式，这里定义了三次。

![image-20220904160217573](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/image-20220904160217573.png)

​	在定义了数据信息的格式之后，就可以使用中断传输的方式传输HID的信息了。

### HID信息传输

​	鸽了，想做的时候再分析吧（
