---
layout: post
title: "XML知识整理"
date: 2023-01-09
tags: [xml]
comments: true
toc: true
author: JumpWang


---

> 你是否有很多xml放不下？做人要潇洒一点。喜欢C语言，未必要写它一辈子的。我喜欢html，未必要给它写好看的css。我看不懂xml，难道直接去叫二进制过来，你让我反编译一下。~~诶，这个好像还真可以。~~
>
> 咳咳，总之，这是一篇简单的关于xml的博客。

​	总能在各个项目里看到xml的身影，我的印象中xml是个长得和html差不多的玩意，可具体他是什么又说不清楚。于是打算系统学习下xml的相关知识。以及本篇文章不会讨论语法细节，主要围绕作用功能做整理~~（大概标题可以换成：如何看懂xml？~~。这篇博客权当是一个学习笔记吧。

## 概念与本质

​	[XML](www.rfc-editor.org/rfc/rfc7303.txt) 指可扩展标记语言（EXtensible *M*arkup *L*anguage），是一种用于存储、传输和重建任意数据的标记语言和文件格式。它定义了一组规则，用于以人类可读和机器可读的格式对文档进行编码。万维网联盟 [1998年](https://www.w3.org/TR/1998/REC-xml-19980210.html) 的 [XML 1.0 规范](www.w3.org/TR/REC-xml)和[其他几个相关规范](https://web.archive.org/web/20130424125723/http://www.dblab.ntua.gr/~bikakis/XML%20and%20Semantic%20Web%20W3C%20Standards%20Timeline-History.pdf)——它们都是免费的开放标准——定义了 XML。

​	以上来自维基百科。

​	w3school有两句话对我可谓醍醐灌顶，一语惊醒梦中人。

- **HTML 被设计用来显示数据。**
- **XML 被设计用来传输和存储数据。**

## 基础

### xml自我描述

​	xml可以非常简单的用一个标签描述自己，最常见的应该如下所示：

​	这句经常出现在文件开头的一行代码表示这个文件是xml文件，版本为1.0，编码方式为UTF-8

```xml
<?xml version="1.0" encoding="UTF-8"?>
```

### xml元素与属性

​	让我们嫖个例子来说明：

​	下面例子中的`data`就是属性，而`note`、`to`、`from`等等这些则是元素。属性提供关于元素的额外（附加）信息。如果学习过同为标记语言的html则很好理解，但是xml的标签是完全的可扩展可自定义的。

​	元素支持嵌套，嵌套的元素组成了整个xml文档的文档树。如果有写过针对html的爬虫应该很好明白，针对html的爬虫就可以根据html文档树节点进行遍历来获取网页的前端内容。例子中的`note`元素就是文档树的根节点，他的孩子包括`to`、`from`、`heading`、`body`，通过这样的结构就可以了解整个xml文档的内容。

​	从这个文档中我们可以很容易知道，他想传输/存储的数据是一个2008年8月8日写下的便条，便条的内容包括标题、内容、书写方和接收方。

```xml
<note date="08/08/2008">
<to>George</to>
<from>John</from>
<heading>Reminder</heading>
<body>Don't forget the meeting!</body>
</note>
```

​	但是这个代码的写法并不够好。之前说过**XML被设计用来传输和存储数据**。无论是为了方便扩展维护，方便不同的系统、程序对数据进行读取还是方便以嵌套形式展现结构，都应该把数据写在元素内而不是标签内的属性中。便条的日期本身也是十分重要的数据内容。把它写成元素会更好，如下所示：

```xml
<note>
<date>
  <day>08</day>
  <month>08</month>
  <year>2008</year>
</date>
<to>George</to>
<from>John</from>
<heading>Reminder</heading>
<body>Don't forget the meeting!</body>
</note>
```

​	如果我们作为开发者，需要使用xml进行数据存储/传输时，也应保持这样的良好习惯。

### 浏览器与xml

​	在我年少无知的时候，我曾以为，所有的.*ml文件拖到浏览器里，都会像html一样华丽的展示出来，但事实证明我错了。比如xml，你拖进来，它只会显示原本的文本格式文件。因为XML文档不携带有关如何显示数据的信息。他的标签是自定义的，浏览器没法认得，所以一般的浏览器都会显示为源代码。

​	当然，也有办法以特殊的格式在浏览器显示xml，就是使用css文件，不过对于被设计用来传输数据的xml来说，这大概没什么意义。

## 过渡

​	好了，你现在已经明白了xml的本质和基础知识，是一个成熟的xml reader了。下面是一个从AMI公司的redfish相关xml里截取出来的一部分代码。~~相信你一定能很容易的看懂它吧？~~

```xml
<edmx:Edmx xmlns:edmx="http://docs.oasis-open.org/odata/ns/edmx" Version="4.0">

  <edmx:Reference Uri="http://docs.oasis-open.org/odata/odata/v4.0/errata03/csd01/complete/vocabularies/Org.OData.Core.V1.xml">
    <edmx:Include Namespace="Org.OData.Core.V1" Alias="OData"/>
  </edmx:Reference>
  <edmx:Reference Uri="http://docs.oasis-open.org/odata/odata/v4.0/errata03/csd01/complete/vocabularies/Org.OData.Capabilities.V1.xml">
    <edmx:Include Namespace="Org.OData.Capabilities.V1" Alias="Capabilities"/>
  </edmx:Reference>
  <edmx:Reference Uri="http://redfish.dmtf.org/schemas/v1/Resource_v1.xml">
    <edmx:Include Namespace="Resource"/>
    <edmx:Include Namespace="Resource.v1_0_0"/>
  </edmx:Reference>
  
</edmx:Edmx>
```

​	~~反正我当时的感想就是，这都什么玩意~~

## 进阶

### 命名空间

​	上面提到了xml里的标签名是完全可自定义的。那就涉及到了一个问题：命名冲突。在同一个xml里，我想给一个标签起名叫table来表示桌子，一个标签起名也叫做table来表示定义一个表格（这是html中的table含义），该怎么办呢？

#### 增加前缀

​	最容易想到的办法自然是为不同的名字划分各自的“定义域”。具体表现为为他们添加前缀。这里还是嫖个例子来说明，如下所示：

​	h前缀的table表示html中的表格的意思

```xml
<h:table>
   <h:tr>
   <h:td>Apples</h:td>
   <h:td>Bananas</h:td>
   </h:tr>
</h:table>
```

​	f前缀的table表示这是一个桌子

```xml
<f:table>
   <f:name>African Coffee Table</f:name>
   <f:width>80</f:width>
   <f:length>120</f:length>
</f:table>
```

​	可以简单的理解为同一个前缀就是同一个命名空间。

#### 命名空间

​	xml存在namespace属性来定义命名空间，即`xmlns`

​	当命名空间被定义在元素的开始标签中时，所有带有相同前缀的子元素都会与同一个命名空间相关联。

​	使用如下语法：

```xml
xmlns:namespace-prefix="namespaceURI"
```

​	我们可以注意到，引号内的是一个命名空间的URI，但为什么这里要放一个URI呢？

> 用于标示命名空间的地址不会被解析器用于查找信息。其惟一的作用是赋予命名空间一个惟一的名称。不过，很多公司常常会作为指针来使用命名空间指向实际存在的网页，这个网页包含关于命名空间的信息。

​	下面看一个命名空间的例子以及之前的过渡部分的代码（这回是不是好懂多了？）：

```xml
<h:table xmlns:h="http://www.w3.org/TR/html4/">
   <h:tr>
   <h:td>Apples</h:td>
   <h:td>Bananas</h:td>
   </h:tr>
</h:table>
```

```xml
<edmx:Edmx xmlns:edmx="http://docs.oasis-open.org/odata/ns/edmx" Version="4.0">

  <edmx:Reference Uri="http://docs.oasis-open.org/odata/odata/v4.0/errata03/csd01/complete/vocabularies/Org.OData.Core.V1.xml">
    <edmx:Include Namespace="Org.OData.Core.V1" Alias="OData"/>
  </edmx:Reference>
  <edmx:Reference Uri="http://docs.oasis-open.org/odata/odata/v4.0/errata03/csd01/complete/vocabularies/Org.OData.Capabilities.V1.xml">
    <edmx:Include Namespace="Org.OData.Capabilities.V1" Alias="Capabilities"/>
  </edmx:Reference>
  <edmx:Reference Uri="http://redfish.dmtf.org/schemas/v1/Resource_v1.xml">
    <edmx:Include Namespace="Resource"/>
    <edmx:Include Namespace="Resource.v1_0_0"/>
  </edmx:Reference>
  
</edmx:Edmx>
```

#### 默认命名空间

​	在使用命名空间的时候，每一个标签的前面都要加一个冒号，还有前缀名，写起来是否有些过于麻烦了呢？

​	所以有了默认命名空间，在元素的开始标签中进行声明之后，所有的子元素与该元素为同一命名空间，省略了命名空间的名称

​	默认命名空间使用如下语法：

```xml
xmlns="namespaceURI"
```

​	例子如下：

```xml
<table xmlns="http://www.w3.org/TR/html4/">
   <tr>
   <td>Apples</td>
   <td>Bananas</td>
   </tr>
</table>
```

### CDATA

​	如果看到了CDATA，不用惊慌，CDATA十分简单。CDATA 部分由 "**<![CDATA[**" 开始，由 "**]]>**" 结束，其中的所有内容都会被解析器忽略。

## 参考

https://www.w3school.com.cn/xml/index.asp

