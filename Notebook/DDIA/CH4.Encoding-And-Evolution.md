# 0.简介

当服务功能发生变化，底层的数据可能也要跟着变化，从而数据的展示方式也可能发生改变。

早在第二章的时候就讨论过数据模型与代码升级的关联（用ORM映射还是松耦合的文档式结构）。本章节从具体的数据存储格式来讨论不同的编码方式、编码兼容方面的问题。



# 1.背景

数据格式兼容问题出现在：

1. 服务端程序往往采用灰度上线，也就是先把新版程序发布到少数几个节点上，没问题再发几台，渐进发布。灰度期间，运行环境有不同版本数据格式的服务交互；
2. 客户端版本升级时机不可控；

这就意味着新老版本的程序和数据在一段时间内是共存的。为了你的应用能够正常运行，你需要做两个方向的兼容。

- *向后兼容*（实现容易，因为你知道老版数据的组织格式，所以可以针对他做特殊处理）
  新版程序可以读写老版数据
- *向前兼容*（实现困难）
  老版程序可以读写新版数据



# 2.数据编码格式

一个数据最少有两种存储表达方式

- 内存中，数据是以对象，结构体，队列，hash表，树的结构表达存储的。不同的结构的目标是能够高效的让CPU找到需要的数据并对之操作。
- 当你把一份数据写入磁盘或者通过互联网传送的时候，你需要把他转化成字节流。这个时候，例如指针对于其他进程来说就没办法理解了，所以可能与他在内存当中的结构全然不同。

所以当从内存到IO时，数据就需要进行一次编码，反过来，就需要一次解码。

## 2.1语言相关格式

最常见的就是java内置的序列化，性能和使用场景都十分有限，这些编码方式主要有以下问题：

* 编解码必须使用相同语言
* 编解码安全问题
* 数据格式版本兼容十分不方便
* 编码性能普遍较差

## 2.2 JSON,XML,CSV

这些编码格式完全独立于编程语言的文本格式来存储和表示数据，但是依然存在一些问题：

* XML和CSV无法区分数值和字符串，JSON可以区分，但是无法支持数值精度；
* JSON和XML不支持二进制裸数据（使用Base64或者自定义编码可以绕过，但带来的是数据体积膨胀和二次编解码的损耗）；
* XML、JSON都属于schema-on-read，需要硬编码来解析数据；
* CSV难以处理拥有分隔符的列数据（使用转义字符可以绕过）

尽管JSON和XML都有很多变种来弥补上面的缺陷，但是单从编码的数据是无法反推出对象的原生结构。（就好像之前遇到的JSON自动把长整型转化为整型数的case）

## 2.3 Thrift和ProtoBuf

Thrift和Protocol Buffers都需要给他们编码的对象定义一个schema，称为接口定义语言( interface definition language, IDL)

```
Thrift
struct Person {
  1: required string userName,
  2: optional i64 favoriteNumber,
  3: optional list<string> interests
}

Protocol Buffer
message Person {
  required string user_name = 1;
  optional int64 favorite_number = 2;
  repeated string interests = 3;
}
```

上面的1，2，3表示的是字段的tag值，服务端和客户端根据tag和字段类型进行映射，达到精准编解码的效果。

<img src="/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/Protobuf-data-format.png" alt="Protobuf-data-format" style="zoom:80%;" />

### 2.3.1 schema兼容性

每个属性用它的tag编号来作为唯一的标记，用一个type声明他的类型。不管是向前兼容还是向后兼容，一旦把一个字段的tag改了，那这个编码的数据就不可用了。

* 向后兼容

  老版本数据不存在一些字段，只要这些字段是非required，新版本代码解析就不会有问题。

* 向前兼容

  当你需要加字段的时候，就可以直接在schema里面加，然后给他一个全新的数字作为field tag。如果老代码看到这个他不认识的tag，他可以直接忽略掉这个字段。而数据编码里面有类型和长度信息，程序就可以直接跳过这个字段的所有数据继续解析。这就保证了向前兼容，老版本代码解析新版本数据。

### 2.3.2 数据类型兼容性

如果一个字段类型发生变化，可能会出现：

1. 数据截断，比如字段从int改成long型，老版本代码解析新版本数据时就可能出现数据截断；
2. 从容器到单值（比如去掉repeated），这时候新版本代码解析老版本数据，就会出现数据丢失。

## 2.4 Avro

Avro也需要有一个Avro IDL来描述对象的数据结构，不同于Thrift和Protobuf的是，Avro区分writer-schema和reader-schema两套数据结构。

<img src="/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/Avro-data-format.png" alt="Avro-data-format" style="zoom:80%;" />

上图可以看出，Avro的数据中，既没有tag也没有type，仅仅只有数据长度、null/unsigned标识和裸数据。编解码完全按使用的schema中字段顺序和类型。得益于这个特性，Avro schema可以动态的改变，而无需根据IDL生成接口文件。

### 2.4.1 schema兼容性

因为writer-schema和reader-schema不需要完全一致，只需要两者兼容即可。

* 写入的字段writer-schema没有则忽略
* 根据reader-schema读取的字段不存在，则赋予默认值。

对于Avro来说，向前兼容就意味着你可以用新版schema写并且用老版schema读，向后兼容就意味着你可以用老版schema写用新版schema读。为了保证兼容性，就要求只能增删有默认值的属性。例如你加了一个字段，新版的读程序在读到老数据时，因为数据中没有这个字段会给他一个默认值，这样就可以向后兼容。如果删一个字段，同样老版读程序在读到新数据时，也会给他一个默认值，就做到向前兼容。



# 3.不同使用场景下编码格式的选择

<font color='red'>这里篇幅比较长，暂时跳过</font>





