---
layout: post
title:  网络：数据流
description: "网络数据流相关概念"
modified: 2015-11-06
tags: [Network]
---

## 数据流概念
通过网络传输数据，或者对文件数据进行操作时，都需要先将数据转换为数据流。**数据流**(Stream)是对串行传输数据的一种抽象表示。
典型的数据流和某个外部数据源相关，数据源可以是文件、外部设备、内存、网络套接字等。根据数据源不同，.NET提供多个从Stream类派生的子类来对不同的数据源提供支持，每个类都代表了一种具体的数据流类型。例如和磁盘文件相关的文件流FileStream，和Socket相关的NetworkStream，和内存相关的MemoryStream。

流提供3种基本操作：<br>
写入：将数据从内存缓冲区传输到外部源。<br>
读取：将数据从外部源传输到内存缓冲区。<br>
查找：重新设置流的当前位置，以便随机读写。需要注意的是，并不是所有的流类型都能够支持查找，如网络流没有当前位置的概念，因此不支持查找功能。

Stream类及其派生类都提供了Read和Write方法，可支持在字节级别上对数据进行读写。Read方法从当前流读取字节序列，Write方法向当前流中写入字节序列。但是仅支持字节级别的数据处理会给开发人员带来不便。因此，除了Stream及其派生类的读写方法之外，.NET框架同样提供了其他多种支持流读写的类。

如[BinaryReader](https://msdn.microsoft.com/zh-cn/library/system.io.binaryreader(v=vs.110).aspx)、[BinaryWriter](https://msdn.microsoft.com/zh-cn/library/system.io.binarywriter(v=vs.110).aspx)，[StreamReader](https://msdn.microsoft.com/zh-cn/library/system.io.streamreader(v=vs.110).aspx)、[StreamWriter](https://msdn.microsoft.com/zh-cn/library/system.io.streamwriter(v=vs.110).aspx).

## 文件流 FileStream
FileStream类的实例代表一个文件流。使用FileStream类可以对文件系统上的文件进行读取、写入、打开和关闭操作。
使用文件流首先要获取实例，有许多方法可以实现。
读取文件，可以使用FileStream的Read方法，也可以创建BinaryReader或StreamReader来读取。要注意，有时磁盘文件很大，如果直接将文件所有数据读入内存是很危险的，所以对文件读取时，一般创建一个较小字节的字节数组作为缓冲区，分块循环读取。
写入文件，可以使用FileStream的Write方法，也可以创建BinaryWriter或StreamWriter来读取。

[MSDN FileStream](https://msdn.microsoft.com/zh-cn/library/system.io.filestream(v=vs.110).aspx) 类。

## 内存流 MemoryStream
和文件流不同，MemoryStream类表示的是保存在内存中的数据流，由内存流封装的数据可以在内存中直接访问。内存流一般用于暂时**缓存数据**，以降低应用程序中对临时缓冲区和临时文件的需要。

既然字节数组也在内存中存储，为什么还要引人内存流的概念？这是因为内存流相对于字节数组而言，具有流特有的特性，并且容量可自动增长。在数据加密以及对长度不定的数据进行缓存等场合，使用内存流比较方便。

Memorystream类的构造函数有多种重载形式，常用的有以下3种:

1. MemoryStream( )。该构造函数初始分配的容量大小为0，随着数据的不断写人，其容量可以不断地自动扩展。一般用于数据内容及大小都不确定的场合。
2. MemoryStream(Byte[])。 通过该构造函数获取的MemoryStream实例根据Byte类型的字节数组进行初始化，并且实例的容量大小固定为字数组的长度。由于实例的容量不能扩展，该构造函数一般用于数据不发生变化的场合。
3. MemoryStream(int capacity)。初始容量大小为capacity，并且可以扩充容量。

Memorystream的Read和Write与文件流类似。不再赘述。
Memorystream支持对数据流的查找和随机访问。

[MSDN Memorystream](https://msdn.microsoft.com/zh-cn/library/system.io.memorystream(v=vs.110).aspx)类。

## 网络流 NetworkStream
在网络上传输数据时，使用的是网络流（NetworkStream）。网络流的意思是数据在网络的各个位置之间是以连续的字节形式传输的。为了处理这种流，C#在System.Net.Sockets命名空间中提供了一个NetworkStream类，用于发送和接收网络数据。
可以将NetworkStream看作在数据来源端和接收端之间架设了一个数据通道，这样一来，读取和写入数据就可以针对这个通道来进行。需要注意的是，NetworkStream类仅支持面向连接的套接字。

对于NetworkStream流，写人操作是指从来源端内存缓冲区到网络上的数据传输；读取操作是从网络上到接收端内存缓冲区（如字节数组）的数据传输。

![]({{ site.url }}/images/post/Network/NetworkStream.png)

一旦构造了一个NetworkStream对象，就可以使用它进行网络数据发送和接收。

[MSDN NetworkStream](https://msdn.microsoft.com/zh-cn/library/system.net.sockets.networkstream(v=vs.110).aspx) 类。