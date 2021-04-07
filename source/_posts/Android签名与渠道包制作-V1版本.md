title: Android签名与渠道包制作-V1版本
date: 2021-04-07 19:47:25
tags:
    - 技术相关
    - Android
---

系列文章:

- [Android签名与渠道包制作-V1版本](https://blog.islinjw.cn/2021/04/07/Android%E7%AD%BE%E5%90%8D%E4%B8%8E%E6%B8%A0%E9%81%93%E5%8C%85%E5%88%B6%E4%BD%9C-V1%E7%89%88%E6%9C%AC/)
- [Android签名与渠道包制作-V2/V3签名原理](https://blog.islinjw.cn/2021/04/07/Android%E7%AD%BE%E5%90%8D%E4%B8%8E%E6%B8%A0%E9%81%93%E5%8C%85%E5%88%B6%E4%BD%9C-V2-V3%E7%AD%BE%E5%90%8D%E5%8E%9F%E7%90%86/)




偶然发现安卓的签名V3已经出到了版本，想想自己其实也没有太深入了解过v1、v2。本着查漏补缺的想法把三个版本的原理都过了一遍，并且利用签名的原理手撸了渠道包制作的demo。这系列的文章就带大家深入了解下各个版本的签名和渠道包制作原理。

这篇我们先来看看V1版本的原理。

# V1签名原理

首先我们要知道用v1签名的apk包其实就是一个普通的zip压缩包，我们将后缀改成.zip就可以直接解压。解压出来可以在META-INF目录下看到MANIFEST.MF、CERT.SF、CERT.RSA这三个文件，V1签名就是靠的这三个文件来验证的。

## MANIFEST.MF

MANIFEST.MF长这个样子，它记录了apk所有原始文件的数据摘要的Base64编码:

```
Manifest-Version: 1.0
Built-By: Generated-by-ADT
Created-By: Android Gradle 3.5.1

Name: AndroidManifest.xml
SHA-256-Digest: 6gizONW6AQK41R0kXhGh+M60wBxPA06WFrq5KSWrB24=

Name: META-INF/androidx.appcompat_appcompat.version
SHA-256-Digest: n9KGQtOsoZHlx/wjg8/W+rsqrIdD8Cnau4mJrFhOMbw=

...
```

正如我们所说，apk是个普通的压缩包，我们解压完修改里面的内容(如图片)再压缩回去，它仍然符合apk文件格式可以用于安装。但是V1签名在安装的时候会用MANIFEST.MF去检查原始文件是否被修改，如果被修改就拒绝安装。

## CERT.SF

当然我们也可以将MANIFEST.MF一起修改了，但是安卓还会通过CERT.SF去检查MANIFEST.MF是否被修改。

CERT.SF长这样:

```
Signature-Version: 1.0
Created-By: 1.0 (Android)
SHA-256-Digest-Manifest: aed+nGnbmO5m79Dy1aNQ68aTFC9N5EyZj8kOeE56yyU=

Name: AndroidManifest.xml
SHA-256-Digest: QA9D/hXYs4aCJcZ4nZ8kLP2RnPn/kw15girRaw7xdng=

Name: META-INF/androidx.appcompat_appcompat.version
SHA-256-Digest: ABbgKP0s08CVeuJ5ZMlIZx/AvJtb1QhNA0ffeXfCaHk=

Name: META-INF/androidx.arch.core_core-runtime.version
SHA-256-Digest: PjygIQMN5T6nIKT/hi5PFaxVcEB+W20fr4f0g2n7jrg=

...
```

它将MANIFEST.MF整个文件和里面的每一项的摘要信息又做一次SHA摘要和Base64编码记录起来。例如CERT.SF第一个AndroidManifest.xml的SHA-256-Digest代表的其实是下面内容SHA摘要的Base64编码:

```
Name: AndroidManifest.xml
SHA-256-Digest: 6gizONW6AQK41R0kXhGh+M60wBxPA06WFrq5KSWrB24=
\r\n
```

PS: 最后一行的\r\n也是要参与计算的。

## CERT.RSA

同样的道理在修改apk的时候我们也可以将MANIFEST.MF和CERT.SF一并修改了。这个时候就轮到CERT.RSA出马了。

进行V1签名的时候会先计算CERT.SF的摘要，然后用开发者的私钥计算数字签名，然后将数字签名、开发者公钥等信息保存到CERT.RSA，在安装的时候就能进行验证。如果没有私钥，修改完CERT.SF就没有办法同步修改CERT.RSA。

## 签名与校验流程

通过上面的介绍我们能总结出V1版本的签名和校验流程:

签名流程:

1. 计算每个原始文件的SHA摘要，用Base64编码保存到MANIFEST.MF
2. 对MANIFEST.MF的整个文件和里面的每一项信息再进行SHA摘要，用Base64编码保存到CERT.SF
3. 计算CERT.SF的摘要并使用开发者的私钥加密计算出数字签名，将该数字签名和开发者公钥等信息保存到CERT.RSA

验证流程:

1. 在CERT.RSA读取公钥和CERT.SF的数字签名，计算CERT.SF的摘要
2. 验证CERT.SF是否被修改
3. 通过CERT.SF验证MANIFEST.MF是否被修改
4. 通过MANIFEST.MF验证原始文件是否被修改

# 渠道包原理

由于国内的应用市场众多，一般需要打多个渠道包上传，这些渠道包会保存该渠道的一些信息。虽然我们可以通过gradle的productFlavors去编译多个包，但是由于这种机制没生成一个渠道包都要走一遍编译流程，耗时比较多。而且一般会生成不同的BuildConfig.java类导致dex不同，如果使用Tinker需要对不同的渠道包都单独做差异包去热修复。

所以一般都不会用这种方式去打渠道包，而是在编译完之后在apk里面插入渠道信息。

刚刚我们也有讲到V1签名会对apk里面的文件进行校验，但是这里有个漏洞就是它是对原始文件进行校验，对整个apk包没有做校验。所以我们可以在apk包中插入渠道信息。

## zip包格式

使用将数据直接插入apk文件的方式，我们先要了解下apk(也就是zip包)的[文件格式](https://pkware.cachefly.net/webdocs/APPNOTE/APPNOTE-6.2.0.txt):

{% img /Android签名与渠道包制作V1版本/1.png %}

它主要分成了上面的三个部分，而我们的突破口就在最后一部分，我们来看看这部分的详细格式:

| 内容                                                         | 大小            |
| ------------------------------------------------------------ | --------------- |
| end of central dir signature (0x06054b50)                    | 4 bytes         |
| number of this disk                                          | 2 bytes         |
| number of the disk with the start of the central directory   | 2 bytes         |
| total number of entries in the central directory on this disk | 2 bytes         |
| total number of entries in the central directory             | 2 bytes         |
| size of the central directory                                | 4 bytes         |
| offset of start of central directory with respect to the starting disk number | 4 bytes         |
| .ZIP file comment length                                     | 2 bytes         |
| .ZIP file comment                                            | (variable size) |

这部分我们简称eocd，它以一个魔数0x06054b50打头，后面带了一些zip包的描述。其中对我们最重要的是最后的.ZIP file comment length和.ZIP file comment。

zip包是可以在末尾携带描述信息的。描述信息的长度在.ZIP file comment length字段中获取。所以我们可以将渠道信息写到.ZIP file comment里。我这里参考[VasDolly]([https://github.com/Tencent/VasDolly/wiki/VasDolly%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86](https://github.com/Tencent/VasDolly/wiki/VasDolly实现原理))的实现原理将渠道信息格式定义成下面的样子插入到apk包的最末尾:

{% img /Android签名与渠道包制作V1版本/2.png %}

于是我们在运行的时候就能通过读取apk包的结尾4个字节看看是否能读到我们定义的魔数判断有无渠道信息，如果有的话往前两个字节读渠道信息的长度，最后根据长度再往前读取渠道信息。

## 渠道信息的写入

这边实现了个Demo，我们直接来看看代码:

```java
public boolean addChannelInfo(String srcApk, String outputApk, String channelInfo) {
    RandomAccessFile zipFile = null;
    FileOutputStream fos = null;
    FileChannel srcChannel = null;
    FileChannel dstChannel = null;
    try {
        zipFile = new RandomAccessFile(new File(srcApk), "r");
        srcChannel = zipFile.getChannel();

        fos = new FileOutputStream(outputApk);
        dstChannel = fos.getChannel();

        // 查找eocd
        ByteBuffer originEocd = Utils.findEocd(srcChannel);
        if (originEocd == null) {
            return false;
        }

        // 往eocd插入渠道信息得到新的eocd
        ByteBuffer newEocd = addChannelInfo(originEocd, channelInfo);

        // eocd前面的数据是没有改到的,直接拷贝就好
        Utils.copyByLength(srcChannel, dstChannel, zipFile.length() - originEocd.capacity());

        // 往后插入新的eocd
        dstChannel.write(newEocd);
    } catch (Exception e) {
        e.printStackTrace();
        return false;
    } finally {
        Utils.safeClose(srcChannel, zipFile, dstChannel, fos);
    }
    return true;
}
```

eocd的读取很简单，从后往前查找eocd魔数即可:

```java
public static ByteBuffer findEocd(FileChannel zipFile) throws IOException {
    // end of central directory record 是整个zip包的结尾
    // 而且它以0x06054b50这个魔数做起始,所以只需从后往前遍历找到这个魔数,即可截取整个EOCD
    //
    // [zip包其余内容]      ...
    //
    // [EOCD]              end of central dir signature (0x06054b50)
    //                     eocd其余部分

    try {
        if (zipFile.size() < Utils.EOCD_MIN_LENGTH) {
            return null;
        }

        int length = (int) Math.min(Utils.EOCD_MAX_LENGTH, zipFile.size());
        ByteBuffer buffer = ByteBuffer.allocate(length);
        buffer.order(ByteOrder.LITTLE_ENDIAN);

        zipFile.read(buffer, zipFile.size() - length);

        for (int i = length - Utils.EOCD_MIN_LENGTH; i >= 0; i--) {
            if (buffer.getInt(i) == Utils.EOCD_SIG) {
                buffer.position(i);
                return buffer.slice().order(ByteOrder.LITTLE_ENDIAN);
            }
        }
        System.out.println("return null");
        return null;
    } finally {
        zipFile.position(0);
    }
}
```

如果apk本身没有带描述，我们主需要直接读最后的22个字节就好，但是为了兼容带描述的情况，我们还是通过查找魔数的方式定位eocd。

.ZIP file comment length只有2字节,所以描述长度最多有0xffff,然后加上eocd前固定的22个字节就得到eocd可能的最大长度

```java
public static final int EOCD_MAX_LENGTH = 0xffff + 22;
```

所以我们直接从apk最后读取这么多个字节去遍历就好。

查到到eocd之后我们在最后插入渠道信息，然后同步修改.ZIP file comment length字段:

```java
private static ByteBuffer addChannelInfo(ByteBuffer eocd, String channelInfo) {
    // end of central directory record 的格式如下:
    //
    // end of central dir signature                                                    4 bytes  (0x06054b50)
    // number of this disk                                                             2 bytes
    // number of the disk with the start of the central directory                      2 bytes
    // total number of entries in the central directory on this disk                   2 bytes
    // total number of entries in the central directory                                2 bytes
    // size of the central directory                                                   4 bytes
    // offset of start of central directory with respect to the starting disk number   4 bytes
    // .ZIP file comment length                                                        2 bytes
    // .ZIP file comment                                                               (variable size)
    //
    // 我们可以在.ZIP file comment里面插入渠道信息块:
    //
    // 渠道信息      大小记录在[渠道信息长度]中
    // 渠道信息长度  2字节
    // 魔数         4字节
    //
    // 魔数放在最后面方便我们读取判断是否有渠道信息

    short infoLength = (short) channelInfo.getBytes().length;
    short channelBlockSize = (short) (infoLength // 渠道信息
            + Short.BYTES      // 渠道信息长度
            + Integer.BYTES);  // 渠道信息魔数
    ByteBuffer buffer = ByteBuffer.allocate(eocd.capacity() + channelBlockSize);
    buffer.order(ByteOrder.LITTLE_ENDIAN);

    // eocd前面部分的数据我们没有改动,直接拷贝就好
    byte[] bytes = new byte[Utils.EOCD_MIN_LENGTH - Utils.EOCD_SIZE_OF_COMMENT_LENGTH];
    eocd.get(bytes);
    buffer.put(bytes);

    // 由于插入了渠道信息块,zip包的注释长度需要相应的增加
    buffer.putShort((short) (eocd.getShort() + channelBlockSize));

    // 拷贝原本的zip包注释
    eocd.position(Utils.EOCD_MIN_LENGTH);
    buffer.put(eocd);

    // 插入渠道包信息块
    buffer.put(channelInfo.getBytes());     // 渠道信息
    buffer.putShort(infoLength);            // 渠道信息长度
    buffer.putInt(Utils.CHANNEL_INFO_SIG);  // 魔数

    buffer.flip();
    return buffer;
}
```

## 渠道信息的读取

讲完渠道信息的写入，我们再来看看运行的时候怎么去读取渠道信息:

```java
public String getChannelInfo(Context context) {
    String apkPath = Utils.getApkPath(context);
    if (apkPath == null) {
        return null;
    }
    RandomAccessFile apk = null;
    try {
        apk = new RandomAccessFile(apkPath, "r");

        // 读取apk的结尾4字节看看是否为渠道信息魔数判断是否有渠道信息
        long sigPosition = apk.length() - Integer.BYTES;
        int sig = Utils.readInt(apk, sigPosition);
        if (sig != Utils.CHANNEL_INFO_SIG) {
            return null;
        }

        // 再往前读两个字节获取渠道信息的长度
        long lengthPosition = sigPosition - Short.BYTES;
        short length = Utils.readShort(apk, lengthPosition);
        if (length <= 0) {
            return null;
        }

        // 根据长度读取渠道信息
        long infoPosition = lengthPosition - length;
        return Utils.readString(apk, infoPosition, length);
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        Utils.safeClose(apk);
    }

    return null;
}
```

流程很简单:

1. 判断apk结尾4个字节是否为渠道信息魔数
2. 获取渠道信息长度
3. 读取渠道信息

# 完整代码

完整的demo已经上传到[Github](https://github.com/bluesky466/ChannelInfoHelper),我将添加渠道信息的操作放到了[单元测试](https://github.com/bluesky466/ChannelInfoHelper/blob/master/app/src/test/java/me/linjw/channelinfohelper/AddChannelInfo.java)里，编译完之后执行插入渠道信息。

