---
layout: docs
title: 关键概念
order: 800
---
<a id="concepts"></a>
# 关键概念

- [开发者、客户、用户](#user)
- [键值对（Key-Value）](#key-value)
- [资源（Resource）](#resource)
- [空间（Bucket）](#bucket)
- [数据处理（Fop）](#fop)
- [数据处理样式（Fop-Style）](#style)

<a id="user"></a>
## 开发者、客户、用户

本文档中会经常提到如下几个角色。

**开发者**是云存储服务的使用者，因此云存储服务的**客户**等同于开发者。  
**用户**是开发者所推出产品的直接使用者，也是云存储服务的间接使用者。  

![用户场景](img/usecase.png "用户场景")

<a id="key-value"></a>
## 键值对（Key-Value）

**键值对（Key-Value，简称KV）**是一个常用的数据结构概念，通常又被称为字典（Dictionary）或映射（Map）。每个存放到该数据结构中的**值（Value）**都对应一个全局唯一的**键（Key）**。该数据结构的特征是以空间换时间，通过键名查询值通常是比较快速的过程。

在存储系统中，键值对和[文件系统（File System）简称FS](/docs/v6/api/overview/glossary.html#fs)是同一个概念。在IT系统演进早期，文件系统比较符合常规的使用场景，令用户可以对大量内容进行有效归类管理，比如桌面操作系统通常会默认创建图片、视频、文档、下载等文件夹。但当系统内的节点（即文件和文件夹）数量达到某个级数后，系统性能将急剧下降到不可用的程度，因此文件系统在设计时一般会对文件夹中可包含的子节点数目设置一个上限。

在设计海量存储系统时，文件系统的两个缺陷就会凸显出来：性能限制、无法横向无限扩展。在这种考虑下，海量存储系统一般设计为键值对模式。键是对应文件的名称，它可以是一个任意字符串（有些设计中可能会保留一些字符），值就是一个具体文件。写入文件时，用户可以自行指定键（比如看起来接近于文件系统的格式：`data/imgs/2011/1/1/img001.jpg`），或者让存储系统自动生成一个唯一的键并返回给上传端。使用者只需要知道下载出口的域名，而无需了解文件具体会被存放到哪个机房的哪个设备，也无需知道具体存放形式。只需在浏览器输入对应的URL访问即可，如：

```
http://imgs.qiniu.com/data/imgs/2011/1/1/img001.jpg
```

<a id="resource"></a>
## 资源（Resource）

**资源**是七牛云存储服务中的逻辑存储单元。对于每一个账号，该账号里存放的每个资源都有唯一的一对**空间（Bucket）**与**键（Key）**，作为识别标识。

资源键名是一个字符串，大部分情况下接近于Linux文件路径的风格，比如：`level1/level2/example1.jpg`。但在七牛云存储中并没有路径的概念，因此这个键名并不表示目录`level1/level2/`下的一个名为`example1.jpg`的文件，而是该键名就是这样一个完整的字符串。因此，与Linux文件路径不同，资源的标识符可以包含任意字符（包括UTF-8编码形式的Unicode字符）。

使用者可以在上传资源时为其指定一个方便管理的键名，通过设计好的前缀来达到类似于文件目录的分类和层次效果。比如对于一个网站的资源，我们可以用如下键名命名资源：

```
index.html
features/index.html
features/feature1.html
features/feature2.html
imgs/features/feature1.png
imgs/features/feature2.png
about.html
```

假设这些资源都位于某个绑定了域名`example.com`的公开空间中，则用户可以在浏览器里输入以下URL访问这些资源：

```
http://www.example.com/features/index.html
``` 

或省略掉`index.html`，如下：

```
http://www.example.com/features/
```

<a id="bucket"></a>
## 空间（Bucket）

**空间**是资源的组织管理单位，一个资源必然位于某个空间中。可以为每个空间设置一系列的属性，以对资源提供合理的管理动作。

常见的属性有如下：

* 将空间设置为**公开**或**私有**，以控制空间内资源的访问权限。
* 设置资源的数据处理样式（Style），以便于用简短方式对资源进行处理。

<a id="fop"></a>
## 数据处理（Fop）

**数据处理**是七牛云存储提供的一种强大的数据处理机制。开发者可以在上传或访问资源时触发数据处理流程，从而得到资源进行按需变换后的结果。

数据处理的执行非常简单，就是在资源请求中加上数据处理动作和参数，比如以下的URL对图片进行了缩放处理，就是调用了一个名为**"imageView2"**的数据处理操作：

```
http://qiniuphotos.qiniudn.com/gogopher.jpg?imageView2/2/w/200/h/200
```

多个数据处理操作可以通过**管道（Pipe）**进行连接，实现级联操作，每一级的输出作为下一级的输入。比如我们可以在一次请求中完成先对图片进行缩放，然后再在图片右下角添加一个透明图片水印，其中缩放和添加水印分别是一个数据处理操作：

```
http://qiniuphotos.qiniudn.com/gogopher.jpg?imageView2/2/w/400
|watermark/1/image/aHR0cDovL3d3dy5iMS5xaW5pdWRuLmNvbS9pbWFnZXMvbG9nby0yLnBuZw==
```

<a id="style"></a>
## 数据处理样式（Fop-Style）

如果觉得`url?<fop1>|<fop2>|<fop3>|<fopN>`这种形式过于冗长，还可以为这些串行的`<fop>`集合定义一个友好别名，之后可以用这个友好别名来取代冗长的指令和参数。我们称这个别名为**数据处理样式**，它是对一个或一组数据处理操作的命名。

还以上一个数据处理的URL为例，看起来非常长且难以理解。我们可以定义一个名为`watermark`的数据处理样式，对应以上的数据处理操作：`imageView2/2/w/400
|watermark/1/image/aHR0cDovL3d3dy5iMS5xaW5pdWRuLmNvbS9pbWFnZXMvbG9nby0yLnBuZw==
`，则我们可以用如下方式进行访问：

```
http://qiniuphotos.qiniudn.com/gogopher.jpg-watermark
```

显然简洁很多也方便很多。

**Note**

上面的 '-' 我们称为数据处理样式的分隔符。
