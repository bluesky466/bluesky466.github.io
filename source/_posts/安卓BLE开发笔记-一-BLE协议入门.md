title: 安卓BLE开发笔记(一) BLE协议入门
date: 2021-09-29 22:26:47
tags:
  - 技术相关
  - Android
---

最近遇到了一个BLE的项目，花时间恶补了下相关的知识，这里记录下来备忘。这篇笔记是纯协议的，先大概了解ble的协议和流程，能帮助我们更好的编码

# Ble设备发现

Ble设备的发现实际上靠的是Advertising(广播)机制。广播也有人管它叫做Beacon,我没有在官方文档里面查找到这个词，但是从网络上的文章来看，它们差不多就是同一个东西。

基于广播发现Ble设备有两种方式:

1. Ble设备设定间隔不断发送广播，手机只要接收到这个广播就能从里面携带的设备信息发现这个设备。同时广播里面也能携带一些特定的数据去实现一些特定的功能，如苹果开发的[*IBeacon*](https://baike.baidu.com/item/iBeacon/13826305?fr=aladdin)。
2. 手机发送Scan Request广播，Ble设备监听到这个设备之后响应一个Scan Response广播

由于这两种方式都基于广播，所以它们的数据格式是一样的。广播会自带一些信息，例如设备的名称、MAC地址等。除了自带的数据之外，我们还能携带一些额外的信息数据。根据[官方](https://www.bluetooth.com/specifications/specs/core-specification/)的[文档](https://www.bluetooth.org/DocMan/handlers/DownloadDoc.ashx?doc_id=521059)，可以看到这个额外数据的具体格式如下:


{% img /安卓BLE开发笔记一/1.png %}

可以看到广播数据里面包含多个AD Structure。每个AD Structure分为两个部分:数据段长度(1字节)+数据段(N字节)。数据段又分为头1个字节的AD Type标识类型和剩余的AD Data具体数据。

注意看最后的Non-significant part,有时候在安卓的回调里面会在byte数组的最后看到一堆的0x0，这个实际上也是定义在协议里面正常的无意义数据，我们直接忽略它们就好。

举个实际的例子，在手机上使用ble搜索应用搜索我司开发的蓝牙设备，查看其广播数据:


{% img /安卓BLE开发笔记一/2.png %}

可以看到广播数据0x0319C703020104030312180C094D41584559455F353146300C16791300000002000000735C，实际有5个AD Structure。

AD Type如上图所说可以去蓝牙协议的[官方](https://www.bluetooth.com/specifications/assigned-numbers/)查看[Generic Access Profile文档](https://btprodspecificationrefs.blob.core.windows.net/assigned-numbers/Assigned%20Number%20Types/Generic%20Access%20Profile.pdf)，可以看到这五个字段类型分别是:

| Data Type Value | Data Type Name                                | Reference for Definition                                     |
| --------------- | --------------------------------------------- | ------------------------------------------------------------ |
| 0x19            | «Appearance»                                  | Bluetooth Core Specification:Core Specification Supplement, Part A, section 1.12 |
| 0x01            | «Flags»                                       | Bluetooth Core Specification:Vol. 3, Part C, section 8.1.3 (v2.1 + EDR, 3.0 + HS and 4.0)Vol. 3, Part C, sections 11.1.3 and 18.1 (v4.0)Core Specification Supplement, Part A, section 1.3 |
| 0x03            | «Complete List of 16-bit Service Class UUIDs» | Bluetooth Core Specification:Vol. 3, Part C, section 8.1.1 (v2.1 + EDR, 3.0 + HS and 4.0)Vol. 3, Part C, sections 11.1.1 and 18.2 (v4.0)Core Specification Supplement, Part A, section 1.1 |
| 0x09            | «Complete Local Name»                         | Bluetooth Core Specification:Vol. 3, Part C, section 8.1.2 (v2.1 + EDR, 3.0 + HS and 4.0)Vol. 3, Part C, sections 11.1.2 and 18.4 (v4.0)Core Specification Supplement, Part A, section 1.2 |
| 0x16            | «Service Data»                                | Bluetooth Core Specification:Vol. 3, Part C, sections 11.1.10 and 18.10 (v4.0) |

根据这里找到的DataType我们又可以去[官方](https://www.bluetooth.com/specifications/specs/)的[Core Specification Supplement 10文档](https://www.bluetooth.com/specifications/specs/core-specification-supplement-10/)查看Value的具体格式。这里我就不展开了。

# GATT协议结构

虽然基于广播的机制我们已经能够实现手机与Ble设备的通信了，但是这种通信能实现的功能比较基础。所以Ble的协议还支持连接之后一对一的通信方式。

Ble的连接都基于 GATT (Generic Attribute Profile) 协议之上，GATT 是一个在蓝牙连接之上的发送和接收很短的数据段的通用规范。

GATT的结构如下:


{% img /安卓BLE开发笔记一/3.png %}

Ble设备里面会提供多个Service，这些Service会提供一些特定的功能。而每个Service里面有会有多个Characteristic，这些Characteristic里面的value实际就是功能具体的属性值。

例如电池服务Battery Service里面就有一个Characteristic叫做Battery Level，手机可以读取这个Characteristic的value值来获取Ble设备当前的电量。

我们来看看实际设备读取出来的数据:


{% img /安卓BLE开发笔记一/4.png %}

可以看到这里有Generic Access、Generic Attribute、Device Information三个Service。而Device Information Service下面又有四个Characteristic

这些Service、Characteristic都会有一个UUID去做标识，例如Generic Access Service的UUID是0x1800。虽然我们这里看到的UUID都是16bit的，但是实际上它们的完整形式应该是0x0000xxxx-0000-1000-8000-00805F9B34FB。中间的xxxx就是截图中显示的16位uuid，即Generic Access Service的完整UUID是0x00001800-0000-1000-8000-00805F9B34FB。

Service的UUID映射表可以到[官方](https://www.bluetooth.com/specifications/assigned-numbers/)的[16-bit UUIDs文档](https://btprodspecificationrefs.blob.core.windows.net/assigned-values/16-bit%20UUID%20Numbers%20Document.pdf)查看。

## Characteristic

### Properties

截图里面的Characteristic的Properties都是READ，代表这个Characteristic是可读取的。它实际上是描述了这个Characteristic可以如何使用。Properties在安卓ble的sdk里面靠一个int类型的变量表示，每一个二进制位都标识了一直能力。我们可以通过按位与的方式判断是否支持:

| **Properties**         | **Value** | **Description**                                              |
| ---------------------- | --------- | ------------------------------------------------------------ |
| Broadcast              | 0x01      | 该Characteristic会被广播出来                                 |
| Read                   | 0x02      | 该Characteristic可读                                         |
| Write Without Response | 0x04      | 该Characteristic可写,写入完成后Ble设备不需要返回响应         |
| Write                  | 0x08      | 该Characteristic可写,写入完成后Ble设备需要返回响应,可以监听到回调 |
| Notify                 | 0x10      | 该Characteristic在改变的时候会发送通知,例如可以监听电量改变的消息去刷新ui |
| Indicate               | 0x20      | 它和Notify类似，只不过主机(手机)在接收到通知的时候会返回一个响应给从机(Ble设备)，这样能够保证通知一定会被接收到 |
| Authenticated          | 0x40      | 该Characteristic需要先配对绑定才能写入                       |
| Extended               | 0x80      | 该Characteristic拥有 Extended Properties                     |

上面的机制实际上我只验证了Read、Write、Notify。其他的我都是根据根据[官方](https://www.bluetooth.com/specifications/specs/core-specification/)的[文档](https://www.bluetooth.org/DocMan/handlers/DownloadDoc.ashx?doc_id=521059)的“Table 3.5: Characteristic Properties bit field”表格自己理解的，可能会有错误，大家可以参考官方文档去理解。


### Descriptors

除了Properties之外，Characteristic还有一个十分重要的数据段叫做Descriptors。一个Characteristic可能有0个或者多个Descriptor去描述它。

例如当一个Characteristic是Notify或者Indicate的时候它会携带一个Client Characteristic Configuration Descriptor(uuid 0x2902)描述当前Characteristic是否打开通知功能。也就是说通知功能是可以通过修改Client Characteristic Configuration Descriptor主动打开或者关闭的，通过Descriptor携带数据的二进制位去标识功能的打开关闭:

| **Configuration** | **Bit** **Number** | **Description**  |
| ----------------- | ------------------ | ---------------- |
| Notification      | 0                  | 打开Notify功能   |
| Indication        | 1                  | 打开Indicate功能 |
|                   | 其他二进制位       | 保留未来使用     |

Descriptor的UUID映射表同样可以到[官方](https://www.bluetooth.com/specifications/assigned-numbers/)的[16-bit UUIDs文档](https://btprodspecificationrefs.blob.core.windows.net/assigned-values/16-bit%20UUID%20Numbers%20Document.pdf)查看。

# MTU

MTU指的是最大传输单元MAXIMUM TRANSMISSION UNIT,表示一次数据传输最多能传多大的数据，我们直接看它的官方说明:


{% img /安卓BLE开发笔记一/5.jpeg %}

可以看到我圈出来的地方,MTU最小需要支持23个字节，但是实际上这23三个字节也不是全部用来放数据，它的头三个字节需要携带操作类型和属性的16位uuid。所以只剩下23-3=20个字节用于传输数据:

{% img /安卓BLE开发笔记一/6.png %}

这里实际上有个坑，安卓默认MTU就是23也就是每次最多只能写入20字节的数据，所以最开始我写入一些比较大的数据的时候直接就失败了。需要先使用requestMtu将mtu设大。
