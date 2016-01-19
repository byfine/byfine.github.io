---
layout: post
title:  网络：C#网络数据编码与解码
description: "C#网络数据编码与解码简介"
modified: 2015-11-05
tags: [Network]
---

在网络通信中，很多情况下通信双方传达的都是字符信息。但是，字符信息并不能直接传递，这些字符信息首先需要被转换成一个字节序列后才能在网络中传输。将字符信息转换为字节序列的过程称为**编码**。当这些字节传送到网络的接收方时，接收方需要反过来将字节序列再转换为字符信息，这种过程称为**解码**。

##字符集
字符(Character)是各种文字和符号的总称，包括各国家文字、标点符号、图形符号、数字等。字符集(Character set)是多个字符的集合。常见的字符集有以下三种：

####ASCII字符集
ASCII（American Standard Code for Information Interchange，美国标准信息交换代码）是基于拉丁字母的一套电脑编码系统，主要用于显示现代英语和其他西欧语言。使用指定的7 位或8 位二进制数组合来表示128 或256 种可能的字符。7 位二进制数主要表示所有的大写和小写字母，数字0 到9、标点符号， 以及在美式英语中使用的特殊控制字符。后128个称为扩展ASCII码。

####非ASCII字符集
由于ASCII字符集是针对英语设计的，当处理汉字等其他字符时，这种编码就不能适应了。
为了解决这些问题，不同的国家和地区指定了自己的编码标准。我国一般使用国标码，常用有GB2312-1980编码和GB18030-2000编码，其中，GB18030编码汉字更多，是我国计算机系统必须遵循的基础性标准之一。

在GB2312编码中，汉字都是采用双字节编码。为了与系统中基本的ASCII字符集区分开，所有汉字编码的每个字节的第一位都是1。GB2312的汉字编码规则为：第一个字节的值在0xB0 ~ 0xF7之间，第二个字节的值在0xA0 ~ 0xFE之间。

GB18030是对GB2312的扩展，其编码长度由2个字节变为1～4个字节。其中包括：

1. 单字节：其值为0 ~ 0X7F。
2. 双字节：第一个字节的值为0X81 ~ 0xFE，第二个字节的值为0X40 ~ 0xFE（不包括0x7F）。
3. 四字节：第一个字节的值为0x81 ~ 0xFE，第二个字节的值为0x30 ~ 0X39，第三个字节的值为0x81 ~ 0xFE，第四个字节的值为0x30～0X39。

可以看出，GB18030的容量非常大，共有码位160万左右。

####Unicode字符集
由于每个国家都拥有独立的编码方式，同一个二进制数字可以被解释成不同的符号。因此，要想打开一个文本文件，就必须知道它的编码方式，否则就可能出现乱码。
为了使国际信息交流更加方便，国际组织制定了Unicode字符集。它为各种语言中的每一个字符规定了统一并且唯一的字符，这种编码只用2个字节就可以表示地球上绝大部分地区的文字。

在C#中，字符默认都是Unicode码，即一个英文字符占两个字节，一个汉字也是两个字节。

Unicode虽然能够表示大部分国家的文字，但由于它比ASCII古用大一倍的空间，这对能用ASCII字符集来表示的字符来说就显得有些浪费。为了解决这个问题，又出现了一些中间格式的字符集，它们被称为通用转换格式，即UTF（Umversal Transformation Format）。目前流行的UTF格式有UTF-8、UTF-16以及UTF-32。

UTF-8是在因特网上使用最广泛的一种UTF格式。它是Unicode的一种变长字符编码，将一个Unicode字符编为1～4个字节组成的UTF-8格式，根据不同的符号而变化字节长度。UTF-8是与字节顺序无关的，它的字节顺序在所有系统中都是一样的，因此这种编码可以使排序变得很容易。

UTF-16将每个码位表示为一个由1 ~ 2个16位整数组成的序列。
UTF-32将每个码位表示为一个32位整数。

UTF-16和UTF-32既可以使用大端字节顺序，又可以使用小端字节顺序。如对于UTF-16，大写字母A(U+0041)，写入到地址0x0000开始的内存：

![]({{ site.url }}/images/post/Network/UTF-Big-Little.png)

##Encoding类

[Encoding](https://msdn.microsoft.com/zh-cn/library/system.text.encoding(v=vs.110).aspx)类位于System.Text命名空间，通过这个类我们可以为不同字符集进行转换以及获取字符集的相关信息 。

####不同编码之间的转换：
利用Encoding.Convert()可以将字节数组从一种编码转换为另一种编码，转换结果为一个byte数组。
原型为：

    public static byte[] Convert( Encoding srcEncoding, Encoding dstEncoding, byte[] bytes )

可以查看[MSDN示例](https://msdn.microsoft.com/zh-cn/library/kdcak6ye(v=vs.110).aspx#Anchor_2)，将使用 Unicode 编码的字符串转换为 ASCII 编码的字符串。


##Encoder 和 Decoder 类

C#中提供了[Encoder](https://msdn.microsoft.com/zh-cn/library/system.text.encoder(v=vs.110).aspx)和[Decoder](https://msdn.microsoft.com/zh-cn/library/system.text.decoder(v=vs.110).aspx)类，分别对字符进行编码和对字节序列进行解码。通过使用它们，我们可以很方便进行对字符和字节序列进行编码和解码操作。

####Encoder：
利用Encoder进行字符编码时，首先要获取Encoder的实例。由于Encoder的构造函数是protected，因此需要通过 Encoding 提供的 GetEncoder 方法获取实例。
获取实例后，就可以使用GetBytes方法将字符编码转换为字节序列。
查看[MSDN示例](https://msdn.microsoft.com/zh-cn/library/5zxk59x5(v=vs.110).aspx#Anchor_3)。

####Decoder：
Decoder类可以将已编码的字节序列解码为字符。
同样需要先用  Encoding 提供的 GetDecoder 方法获取实例。然后利用实例的GetChars方法获取字节序列。
查看[MSDN示例](https://msdn.microsoft.com/zh-cn/library/125z2etb(v=vs.110).aspx#Anchor_3)。