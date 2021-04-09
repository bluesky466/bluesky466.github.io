title: Android签名与渠道包制作-V2/V3渠道包原理
date: 2021-04-09 22:31:53
tags:
    - 技术相关
    - Android
---
系列文章:

- [Android签名与渠道包制作-V1版本](https://blog.islinjw.cn/2021/04/07/Android%E7%AD%BE%E5%90%8D%E4%B8%8E%E6%B8%A0%E9%81%93%E5%8C%85%E5%88%B6%E4%BD%9C-V1%E7%89%88%E6%9C%AC/)
- [Android签名与渠道包制作-V2/V3签名原理](https://blog.islinjw.cn/2021/04/07/Android%E7%AD%BE%E5%90%8D%E4%B8%8E%E6%B8%A0%E9%81%93%E5%8C%85%E5%88%B6%E4%BD%9C-V2-V3%E7%AD%BE%E5%90%8D%E5%8E%9F%E7%90%86/)
- [Android签名与渠道包制作-V2/V3渠道包原理](https://blog.islinjw.cn/2021/04/09/Android%E7%AD%BE%E5%90%8D%E4%B8%8E%E6%B8%A0%E9%81%93%E5%8C%85%E5%88%B6%E4%BD%9C-V2-V3%E6%B8%A0%E9%81%93%E5%8C%85%E5%8E%9F%E7%90%86/)


[上一篇](https://blog.islinjw.cn/2021/04/07/Android%E7%AD%BE%E5%90%8D%E4%B8%8E%E6%B8%A0%E9%81%93%E5%8C%85%E5%88%B6%E4%BD%9C-V2-V3%E7%AD%BE%E5%90%8D%E5%8E%9F%E7%90%86/)文章我们详细描述了V2/V3签名的原理，大概的原理的就是在APK中插入签名块保存签名信息:

{% img /Android签名与渠道包制作V2V3渠道包原理/1.png %}

APK签名块的格式如下:

{% img /Android签名与渠道包制作V2V3渠道包原理/2.png %}

V2的签名数据id为0x7109871a,V3的签名数据id为0xf05368c0。由于校验流程里面对APK本身做了校验，所以V1版本的添加zip file comment的方法就失效了。

但是由于V2/V3签名对这个APK签名块是没有做校验的，所以我们可以添加一个自定义的渠道包信息键值对保存渠道信息。这篇文章就通过分析[Demo](https://github.com/bluesky466/ChannelInfoHelper)代码来讲解V2/V3渠道包的原理。

# 渠道信息写入

基本原理就是渠道信息打包成一个id-value键值对放到键值对序列的最后:

{% img /Android签名与渠道包制作V2V3渠道包原理/3.png %}

插入的代码如下:

```java

@Override
public boolean addChannelInfo(String srcApk, String outputApk, String channelInfo) {
    // [APK签名块]插入在[central directory]之前,而[central directory]的起始位置可以在[EOCD]的socdOffset部分读取
    // 我们在[APK签名块]里面插入渠道信息,会影响到[central directory]的位置,
    // 所以需要同步修改[EOCD]里面的socdOffset
    //
    // [zip包其余内容](不变)          ...
    //
    //                              1. APK签名块大小(不包含自己的8个字节)        8字节
    // [APK签名块](需要插入渠道信息)   2. ID-Value键值对                        大小可变
    //                              3. APK签名块大小(和第1部分相等)             8字节
    //                              4. 魔法数(固定为字符串"APK Sig Block 42")  16字节
    //                                      <--------------------------
    // [central directory](不变)    ...                                |
    //                                                                 |
    //                              end of central dir signature       |
    //                              ...                                |
    // [EOCD](需要修改socdOffset)    socdOffset  ------------------------
    //                              ...
    //

    if (channelInfo == null || channelInfo.isEmpty()) {
        return true;
    }

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
        ByteBuffer eocd = Utils.findEocd(srcChannel);
        if (eocd == null) {
            return false;
        }

        // 获取旧的APK签名块
        long socdOffset = Utils.getSocdOffset(eocd);
        Utils.Pair<Long, ByteBuffer> oldSignV2Block = Utils.getSignV2Block(zipFile, socdOffset);
        if (oldSignV2Block == null) {
            return false;
        }

        // 往APK签名块插入渠道信息,得到新的APK签名块
        ByteBuffer newSignV2Block = addChannelInfo(oldSignV2Block.second, channelInfo);

        // 修改eocd中的socd
        changeSocdOffset(eocd, channelInfo);

        // APK签名块前的数据是没有改过的,可以直接拷贝
        srcChannel.position(0);
        Utils.copyByLength(srcChannel, dstChannel, oldSignV2Block.first);

        // 往后插入新的APK签名块的数据
        dstChannel.write(newSignV2Block);

        // 往后插入[central directory]的数据,这部分也是没有修改的
        srcChannel.position(socdOffset);
        Utils.copyByLength(srcChannel, dstChannel, srcChannel.size() - socdOffset - eocd.capacity());

        // 往后插入修改后的eocd
        eocd.position(0);
        dstChannel.write(eocd);
    } catch (Exception e) {
        e.printStackTrace();
        return false;
    } finally {
        Utils.safeClose(srcChannel, zipFile, dstChannel, fos);
    }
    return true;
}
```

基本流程并不复杂，就是:

1. 通过魔数查找eocd
2. 在eocd中查找central directory的偏移地址socd
3. socd往前读就是APK签名块
4. 在APK签名块中插入渠道信息
5. 由于APK签名块在socd的签名，插入渠道信息后需要将socd往后移
6. 将修改后的各部分重新写入apk

socd的修改很简单，读取原来的值加上渠道信息键值对的长度重新写入即可:

```java
private void changeSocdOffset(ByteBuffer eocd, String channelInfo) {
    // 由于APK签名块在socd offset的前面
    // 而我们又在APK签名块里面插入了渠道信息
    // 所以socd offset应该再往后移动插入的渠道信息键值对的大小


    // 读取原本的socd offset
    eocd.position(Utils.EOCD_POSITION_SOCD_OFFSET);
    int originOffset = eocd.getInt();


    // 键值对格式如下:
    //
    // 键值对长度(不包含自己的8个字节)   8字节
    // ID                            4字节
    // Value                         键值对长度-ID的4字节
    //
    // 所以应该加上键值对长度(8字节)、ID长度(4字节)、渠道信息长度
    eocd.position(Utils.EOCD_POSITION_SOCD_OFFSET);
    eocd.putInt(originOffset + Long.BYTES + Integer.BYTES + channelInfo.getBytes().length);
}
```

往APK签名块中插入渠道信息稍微复杂一点。我们想将渠道信息插入到id-value键值对的最后，但是这里并没有去遍历这个键值对序列，而是用了一种取巧的方式。

由于键值对直接并没有什么指针关系去指定下一个键值对，仅仅只是将将它们排列在一起，所以我们只需要从APK签名块的末尾往前跳过16字节的魔数和8字节的长度信息就能找到插入位置的地址偏移:

{% img /Android签名与渠道包制作V2V3渠道包原理/4.gif %}

由于插入之后APK签名块的长度会增加，所以长度信息需要同步修改，完整的插入代码如下:

```java
private static ByteBuffer addChannelInfo(ByteBuffer oldSignV2BlockSize, String channelInfo) {
    // ID-Value键值对的格式如下:
    //
    // 键值对长度(不包含自己的8个字节)   8字节
    // ID                            4字节
    // Value                         键值对长度-ID的4字节

    // 所以整个ID-Value键的长度应该是 Value长度 + ID长度(4字节) + 键值对长度(8字节)
    long infoLength = channelInfo.getBytes().length;
    long channelBlockRealSize = Long.BYTES + Integer.BYTES + infoLength;

    ByteBuffer buffer = ByteBuffer.allocate((int) (oldSignV2BlockSize.capacity() + channelBlockRealSize));
    buffer.order(ByteOrder.LITTLE_ENDIAN);

    // 先将原本的APK完整拷贝出来
    oldSignV2BlockSize.position(0);
    buffer.put(oldSignV2BlockSize);

    // 读取原本的APK签名块长度
    oldSignV2BlockSize.position(0);
    long originSize = oldSignV2BlockSize.getLong();

    // 该长度要加上插入的渠道信息键值对长度
    buffer.position(0);
    buffer.putLong(originSize + channelBlockRealSize);

    // APK签名块结构如下:
    //
    // 1. APK签名块大小(不包含自己的8个字节)        8字节
    // 2. ID-Value键值对                        大小可变
    // 3. APK签名块大小(和第1部分相等)             8字节
    // 4. 魔法数(固定为字符串"APK Sig Block 42")  16字节

    // 我们把渠道包键值对放到整个APK签名块的最后
    // 所以从后往前减去魔法数的16字节,减去APK签名块大小的8字节
    // 定位到渠道包键值的起始位置
    long magicNumberSize = Utils.SIG_V2_MAGIC_NUMBER.getBytes().length;
    buffer.position((int) (oldSignV2BlockSize.capacity() - magicNumberSize - Long.BYTES));

    // 插入渠道包键值对数据
    buffer.putLong(infoLength + Integer.BYTES);
    buffer.putInt(Utils.CHANNEL_INFO_SIG);
    buffer.put(channelInfo.getBytes());

    // 插入APK签名块长度
    buffer.putLong(originSize + channelBlockRealSize);

    // 插入魔法数
    buffer.put(Utils.SIG_V2_MAGIC_NUMBER.getBytes());

    buffer.flip();
    return buffer;
}
```

# 渠道信息读取

读取部分的逻辑也比较清晰:

1. 通过魔数查找eocd
2. 在eocd中查找central directory的偏移地址socd
3. socd往前读就是APK签名块
4. 跳过APK签名块头8个字节(长度信息)就是第一个id-value键值对
5. 遍历键值对查找渠道信息键值对的id
6. 找到渠道信息键值对之后读取value部分返回即可

```java
public String getChannelInfo(Context context) {
    String apkPath = Utils.getApkPath(context);
    if (apkPath == null) {
        return null;
    }
    RandomAccessFile apk = null;
    try {
        apk = new RandomAccessFile(apkPath, "r");

        // 查找eocd
        ByteBuffer eocd = Utils.findEocd(apk.getChannel());
        if (eocd == null) {
            return null;
        }

        // 获取APK签名块
        Utils.Pair<Long, ByteBuffer> signV2Block = Utils.getSignV2Block(apk, Utils.getSocdOffset(eocd));
        if (signV2Block == null) {
            return null;
        }


        // APK签名块结构如下:
        //
        // 1. APK签名块大小(不包含自己的8个字节)        8字节
        // 2. ID-Value键值对(有多个键值对)            大小可变
        //      2.1 键值对长度(不包含自己的8个字节)     8字节
        //      2.2 ID                              4字节
        //      2.3 Value                           键值对长度-ID的4字节
        // 3. APK签名块大小(和第1部分相等)             8字节
        // 4. 魔法数(固定为字符串"APK Sig Block 42")  16字节


        int id;
        long length,realLength;
        long positionLimit = signV2Block.second.capacity()
                - Long.BYTES                                    // APK签名块大小的长度(8字节)
                - Utils.SIG_V2_MAGIC_NUMBER.getBytes().length;  // 结尾魔数的长度(16字节)

        int position = Long.BYTES; // 跳过开头APK签名块大小的8字节才是第一个ID-Value键值对

        do {
            signV2Block.second.position(position);

            // 读取键值对长度(不包含自己的8个字节)
            length = signV2Block.second.getLong();

            // 键值对长度是不包含长度信息的8个字节的,所以要加上这8个字节
            realLength = Long.BYTES + length;

            // 读取ID
            id = signV2Block.second.getInt();

            // 移动到下一个键值对
            position += realLength;

            // 判断是否找到渠道信息键值对的ID,或者已经遍历完整个APK签名块
        } while (id != Utils.CHANNEL_INFO_SIG && position <= positionLimit);

        if (id == Utils.CHANNEL_INFO_SIG) {
            // 如果可以找到渠道信息键值对,往后读取就可以读到渠道信息
            // 键值对长度是包含ID的四个字节的,要减去
            return Utils.readString(signV2Block.second, (int) (length - Integer.BYTES));
        }
        return null;
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        Utils.safeClose(apk);
    }

    return null;
}
```

# 完整代码

由于V2/V3的原理其实是大致相同的，都是在APK签名块里面插入id-value键值对，所以我们这个做法是能够兼容V2/V3版本的签名的。完整的demo已经上传到[Github](https://github.com/bluesky466/ChannelInfoHelper),我将添加渠道信息的操作放到了[单元测试](https://github.com/bluesky466/ChannelInfoHelper/blob/master/app/src/test/java/me/linjw/channelinfohelper/AddChannelInfo.java)里，编译完之后执行插入渠道信息。

