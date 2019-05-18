title: 可能是全网讲最细的安卓resources.arsc解析教程(一)
date: 2019-05-18 11:58:55
tags:
    - 技术相关
    - Android
---

aapt工具在编译资源会将一些资源或者资源索引打包成resources.arsc。这个文件以二进制数据的形式记录数据，c/c++加载起来特别方便。

了解resources.arsc的结构对理解安卓的资源加载原理有很重要的帮助。

这几天写resources.arsc解析工具时候在网上搜到了不少的资料、博客，但是它们写的都不是特别的详细，都会漏掉一些东西没有提。导致在实现的时候遇到了很多的坑。这里我希望尽量把自己总结出来的东西一步步都列出来，尽量做到只需要看这篇博客就能自己实现一个resources.arsc的解析器。

这个[工具](https://github.com/bluesky466/ResourcesParser)已经在github上开源(使用C++11,已经在mac和ubuntu上报make编译通过正常运行,Windows的同学就只好说声抱歉了)，感兴趣的同学也可以直接下载下来玩玩。

# 总体结构

resources.arsc是以一个个Chunk块的形式组织的,Chunk的头部信息记录了这个Chunk的类型、长度等数据。

从整体上来看，其结构为：资源索引表头部+字符串资源池+N个Package数据块:

{% img /可能是全网讲最细的安卓resources_arsc解析教程一/1.png %}

# 头部数据解析

整个resources.arsc就是一个Chunk块，所以文件的开头就是这个Chunk的头部信息.不过需要注意的是resources.arsc文件采用_小端编码_方式.所以数据应该按字节从后往前读。

头部的结构如下:

头部类型(两个字节)，头部大小(两个字节)，Chunk块大小(四个字节)

我们找一个apk文件，将它后缀改成.zip直接解压，就可以得到resources.arsc，这里实际举个例子，用编辑工具查看到resources.arsc的前8个字节的数据，这里以16进制显示:

```
0200 0c00 acb6 0300
```

首先头部类型的两个字节是0200(每两个16进制数字代表了一个字节)，但是这里是小端序，所以要从后往前读，得到实际的值0x0002。

> 02 00 -> 00 02

resources.arsc内部记录的数据类型在ResourceTypes.h里面定义，我们找到头部类型的枚举，可以查到0x0002对应的类型是RES\_TABLE\_TYPE。返回去看上面的图，可以看到类型的确是RES\_TABLE\_TYPE。

```
enum {                                                                                                          
    RES_NULL_TYPE               = 0x0000,                                                                       
    RES_STRING_POOL_TYPE        = 0x0001,                                                                       
    RES_TABLE_TYPE              = 0x0002,                                                                       
    RES_XML_TYPE                = 0x0003,                                                                       

    // Chunk types in RES_XML_TYPE                                                                              
    RES_XML_FIRST_CHUNK_TYPE    = 0x0100,                                                                       
    RES_XML_START_NAMESPACE_TYPE= 0x0100,                                                                       
    RES_XML_END_NAMESPACE_TYPE  = 0x0101,                                                                       
    RES_XML_START_ELEMENT_TYPE  = 0x0102,                                                                       
    RES_XML_END_ELEMENT_TYPE    = 0x0103,                                                                       
    RES_XML_CDATA_TYPE          = 0x0104,                                                                       
    RES_XML_LAST_CHUNK_TYPE     = 0x017f,                                                                       
    // This contains a uint32_t array mapping strings in the string              
    // pool back to resource identifiers.  It is optional.                       
    RES_XML_RESOURCE_MAP_TYPE   = 0x0180,                                                                       

    // Chunk types in RES_TABLE_TYPE                                                                            
    RES_TABLE_PACKAGE_TYPE      = 0x0200,                                                                       
    RES_TABLE_TYPE_TYPE         = 0x0201,                                                                       
    RES_TABLE_TYPE_SPEC_TYPE    = 0x0202,                                                                       
    RES_TABLE_LIBRARY_TYPE      = 0x0203                                                                        
};
```


接着是头部大小的两个字节0c00(每两个16进制数字代表了一个字节)，从后往前读得到实际值0x000c，也就是说这个头部大小有12个字节

> 0c 00 -> 00 0c

再接着的是Chunk块大小的四个字节acb6 0300(每两个16进制数字代表了一个字节),从后往前读得到实际值0x0003b6ac，转换回十进制是243372，也就是说resources.arsc文件的大小是243372字节。

> ac b6 03 00 -> 00 03 b6 ac

我们可以用wc命令查看这个文件的大小:

```
wc resources.arsc
277   2932 243372 resources.arsc
```

resources.arsc文件这样组织的原因是可以很方便的使用c/c++去加载。

我们在源码里面找到Chunk头部信息的ResChunk\_header结构体:

```
/**                                                                                                             
* Header that appears at the front of every data chunk in a resource.            
*/                                                                                                              
struct ResChunk_header                                                                                          
{                                                                                                               
   // Type identifier for this chunk.  The meaning of this value depends         
   // on the containing chunk.                                                                                  
   uint16_t type;                                                                                               

   // Size of the chunk header (in bytes).  Adding this value to                 
   // the address of the chunk allows you to find its associated data            
   // (if any).                                                                                                 
   uint16_t headerSize;                                                                                         

   // Total size of this chunk (in bytes).  This is the chunkSize plus           
   // the size of any data associated with the chunk.  Adding this value         
   // to the chunk allows you to completely skip its contents (including         
   // any child chunks).  If this value is the same as chunkSize, there is       
   // no data associated with the chunk.                                                                        
   uint32_t size;                                                                                               
};  
```

下面就是show time了，由于普通机器设备上使用的都是小端序号,所以用c语言解析resources.arsc文件的头部信息只要直接填充ResChunk\_header结构体就好了:


```
#include <stdio.h>                                                               
#include <stdlib.h>                                                              
#include <stdint.h>                                                              

struct ResChunk_header {                                                         
    uint16_t type;                                                               
    int16_t headerSize;                                                          
    uint32_t size;                                                               
};                                                                               

int main(int argc, char *argv[]) {                                               
    struct ResChunk_header header;                                               

    FILE* pFile = fopen(argv[1], "rb");                                          
    fread((void*)&header, sizeof(struct ResChunk_header), 1, pFile);             
    fclose(pFile);                                                               

    printf("type:%u, headSize:%u, size:%u\n", header.type, header.headerSize, header.size);
    return 0;                                                                    
}
```

编译后运行,查看打印:

```
./a.out resources.arsc
type:2, headSize:12, size:243372
```

## 头部类型

每个头部类型都会有一个具体的结构体和它对应,例如我们的RES\_TABLE\_TYPE类型对应的就是ResTable\_header:

```
struct ResTable_header                                                           
{                                                                                
    struct ResChunk_header header;                                               

    uint32_t packageCount;                                                       
};
```

这些结构体的第一个成员都是是ResChunk\_header，然后后面才是这个类型的特有数据。所以我们改下代码去读取完整的头部信息:

```
#include <stdio.h>                                                               
#include <stdlib.h>                                                              
#include <stdint.h>                                                              

struct ResChunk_header {                                                         
    uint16_t type;                                                               
    int16_t headerSize;                                                          
    uint32_t size;                                                               
};                                                                               

struct ResTable_header{                                                          
    struct ResChunk_header header;                                               
    uint32_t packageCount;                                                       
};                                                                               

int main(int argc, char *argv[]) {                                               
    uint16_t type;                                                               

    FILE* pFile = fopen(argv[1], "rb");                                          
    fread((void*)&type, sizeof(type), 1, pFile);                                 
    if(type == RES_TABLE_TYPE) {                                                          
        struct ResTable_header header = {0x002};                                 
        fread((void*)(((char*)&header)+2), sizeof(struct ResTable_header)-2, 1, pFile);
        printf("type:%u, headSize:%u, size:%u, packageCount:%u\n",           
                header.header.type,                                              
                header.header.headerSize,                                        
                header.header.size,                                              
                header.packageCount);                                            
    }                                                                            
    fclose(pFile);                                                               
    return 0;                                                                    
}
```

运行得到:

```
./a.out resources.arsc
type:2, headSize:12, size:243372, packageCount:1
```

这里的packageCount指的是resources.arsc里面包含了多少个package的资源,一般只有一个。

到这里，RES\_TABLE\_TYPE的头部信息我们就解析完成了。

# 全局字符串池

{% img /可能是全网讲最细的安卓resources_arsc解析教程一/3.png %}

我们看回上面的图，头部之后紧接着的是Global String Pool。它其实也是一个Chunk，所以也有头部。我们可以用同样的方法去解析:

```
struct ResStringPool_header readResStringPoolHeader(FILE* pFile) {               
    struct ResStringPool_header header;                                          
    uint16_t type;                                                               
    fread((void*)&header, sizeof(struct ResStringPool_header), 1, pFile);        
    printf("type:%u, headSize:%u, size:%u, stringCount:%u, stringStart:%u, styleCount:%u, styleStart:%u\n",
                header.header.type,                                              
                header.header.headerSize,                                        
                header.header.size,                                              
                header.stringCount,                                              
                header.stringsStart,                                             
                header.styleCount,                                               
                header.stylesStart);                                             
    return header;                                                               
}
```

打印如下，这个字符串池里面有1971个字符串:

```
./a.out resources.arsc
type:2, headSize:12, size:243372, packageCount:1
type:1, headSize:28, size:72732, stringCount:1971, stringStart:7912, styleCount:0, styleStart:0
```

接着看图，ResStringPool\_header后面跟着的是

字符串偏移数组+style偏移数组+字符串+style

resources.arsc会把所有的应用里面出现的字符串都放到这个全局字符串池里面，不过style的话我现在还没有理解它的作用，遇到的值都是0，所以这里先忽略。我们只讲字符串。

这个字符串偏移数组其实是一个uint32\_t的数组,记录了每个字符串距离stringStart的偏移，而stringStart就是具体存放字符串的内存的位置距离ResStringPool\_header起始地址的字节数。

需要注意的是，字符串的前两个字节记录了字符串的长度，而且字符串的编码格式由ResStringPool\_header::flags指定(utf-8或者utf-16)

可能这么讲有些抽象，可以结合下面的示意图还有代码理解一下:

{% img /可能是全网讲最细的安卓resources_arsc解析教程一/2.png %}

```
unsigned char* readStringsFromStringPool(FILE* pFile,  struct ResStringPool_header header) {
    uint32_t size = header.header.size - sizeof(struct ResStringPool_header);    
    unsigned char* pData = (unsigned char*)malloc(size);                         
    fread((void*)pData, size, 1, pFile);                                         
    uint32_t* pOffsets = (uint32_t*)pData;                                       

    //stringsStart指的是header的起始地址到字符串起始地址的距离                   
    //pData已经是header末尾的地址了，所以要减去header的大小                      
    char* pStringsStart = pData + header.stringsStart - sizeof(struct ResStringPool_header);

    for(int i = 0 ; i < header.stringCount ; i++) {                              
        //前面两个字节是长度,要跳过                                              
        char* str = pStringsStart + *(pOffsets + i) + 2;                         
        if(header.flags & UTF8_FLAG) {                                           
            printf("%s\n", str);                                                 
        } else {                                                                 
            printUtf16String(str);                                               
        }                                                                        
    }                                                                            

    return pData;                                                                
}                                                                         
```

这里的会把应用里面用到的字符串资源都打印出来，值得注意的是这里的字符串资源指的不仅是我们定义的string标签里的值:

```
...
Delete 鍵
查看全部
瀏覽首頁
空格鍵
與「%s」分享
選擇分享對象
...
```

还包括了资源的路径等其他字符串资源:

```
...
res/anim/abc_fade_in.xml                                                         
res/anim/abc_fade_out.xml                                                        
res/anim/abc_grow_fade_in_from_bottom.xml                                        
res/anim/abc_popup_enter.xml                                                     
res/anim/abc_popup_exit.xml                                                      
res/anim/abc_shrink_fade_out_from_bottom.xml                                     
res/anim/abc_slide_in_bottom.xml                                                 
res/anim/abc_slide_in_top.xml                                                    
res/anim/abc_slide_out_bottom.xml                                                
res/anim/abc_slide_out_top.xml                                                   
res/anim/abc_tooltip_enter.xml                                                   
res/anim/abc_tooltip_exit.xml                                                    
res/color-v23/abc_btn_colored_borderless_text_material.xml                       
res/color-v23/abc_btn_colored_text_material.xml                                  
res/color-v23/abc_color_highlight_material.xml                                   
res/color-v23/abc_tint_btn_checkable.xml          
...
```

# Package资源

{% img /可能是全网讲最细的安卓resources_arsc解析教程一/4.png %}

让我们继续往下读,接下来的就是包内的资源了。resources.arsc可以支持打入多个package的资源,但是一般只会有一个package。

每个package的资源同样打包成一个Chunk,它的头部信息由ResTable\_package结构体表示:


```
int readResTablePackageHeader(FILE* pFile, struct ResTable_package* pHeader) {
    if(fread((void*)pHeader, sizeof(struct ResTable_package), 1, pFile) == 0) {
        return 0;
    }
    printf("type:%u, headSize:%u, size:%u, id:%x, packageName:",
                pHeader->header.type,
                pHeader->header.headerSize,
                pHeader->header.size,
                pHeader->id);
    printUtf16String((char16_t*)pHeader->name);
    return 1;
}
```

头部信息记录了package的id和名字:

```
type:512, headSize:288, size:188068, id:7f, packageName:com.cvte.tv.myapplication
```

## 资源类型字符串池

{% img /可能是全网讲最细的安卓resources_arsc解析教程一/5.png %}

紧接着头部之后的又是一个字符串池,它和之前介绍的全局字符串资源池的结构是一样的,我们可以用同样的方法去读取:

```
struct ResTable_package packageHeader;
while(readResTablePackageHeader(pFile, &packageHeader)) {
    struct ResStringPool_header typeStringPoolHeader = readResStringPoolHeader(pFile);
    unsigned char* pTypeStrings = readStringsFromStringPool(pFile, typeStringPoolHeader);
	...
}
```

这里记录的是这个package里面存储的资源的类型:

```
type:1, headSize:28, size:248, stringCount:12, stringStart:76, styleCount:0, styleStart:0
anim
attr
bool
color
dimen
drawable
id
integer
layout
mipmap
string
style
```

可以看到这个package里面有anim、attr、bool、color、dimen、drawable、id、integer、layout、mipmap、string、style这么多中类型的资源。

## 资源项名称字符串池

{% img /可能是全网讲最细的安卓resources_arsc解析教程一/6.png %}

接下来又是一个字符串资源池,再读一次:

```
struct ResTable_package packageHeader;
while(readResTablePackageHeader(pFile, &packageHeader)) {
    struct ResStringPool_header typeStringPoolHeader = readResStringPoolHeader(pFile);
    unsigned char* pTypeStrings = readStringsFromStringPool(pFile, typeStringPoolHeader);

    struct ResStringPool_header keyStringPoolHeader = readResStringPoolHeader(pFile);
    unsigned char* pKeyStrings = readStringsFromStringPool(pFile, keyStringPoolHeader);
    ...
}
```

这里读出来的就是我们的资源的key:

```
type:1, headSize:28, size:41208, stringCount:1221, stringStart:4912, styleCount:0, styleStart:0
abc_fade_in
abc_fade_out
abc_grow_fade_in_from_bottom
abc_popup_enter
abc_popup_exit
abc_shrink_fade_out_from_bottom
abc_slide_in_bottom
abc_slide_in_top
abc_slide_out_bottom
abc_slide_out_top
abc_tooltip_enter
abc_tooltip_exit
actionBarDivider
actionBarItemBackground
actionBarPopupTheme
actionBarSize
actionBarSplitStyle
actionBarStyle
actionBarTabBarStyle
...
activity_main       <- activity_main在这里
...
app_name            <- app_name在这里
...
```

比如我们的app\_name字符串,就会生成R.string.app\_name,而这个"app\_name"就会出现在资源项名称字符串池里面:

```
<string name="app_name">My Application</string>
```

又或者我们定义了activity\_main.xml,就会生成R.layout.activity\_main,而这个"activity\_main"也会出现在资源项名称字符串池里面。

## 资源

接下来的就是一系列的RES\_TABLE\_TYPE\_SPEC\_TYPE和RES\_TABLE\_TYPE\_TYPE,不过上面的图画的不是很清晰。

Package资源剩下的部分是按资源类型分组的。一个RES\_TABLE\_TYPE\_SPEC\_TYPE跟着多个RES\_TABLE\_TYPE\_TYPE为一组,记录一个类型在不同配置下的资源。

比如我们在前面的资源类型字符串池里面看到有anim、attr、bool、color、dimen、drawable、id、integer、layout、mipmap、string、style,十二种类型的资源,于是就有十二组的RES\_TABLE\_TYPE\_SPEC\_TYPE和RES\_TABLE\_TYPE\_TYPE,而且顺序也是按资源类型字符串池里面的顺序排的:

{% img /可能是全网讲最细的安卓resources_arsc解析教程一/7.png %}

可以看到每一组都是以一个RES\_TABLE\_TYPE\_SPEC\_TYPE开头记录该类型的资源的信息,然后跟着多个RES\_TABLE\_TYPE\_TYPE记录该类型在不同Config下的数据(如color、color-v21、color-v23或者我们更熟悉的string、string-en-US、string-zh-CN、string-zh-TW等)

我们可以实际看看打印:

```
struct ResChunk_header chunkHeader;
uint8_t id;
while(fread((void*)&chunkHeader, sizeof(struct ResChunk_header), 1, pFile)
        && chunkHeader.type != RES_TABLE_PACKAGE_TYPE) {
    fread((void*)&id, sizeof(uint8_t), 1, pFile);
    printf("0x%x, %d\n", chunkHeader.type, id);
    fseek(pFile, chunkHeader.size - sizeof(struct ResChunk_header) - sizeof(uint8_t), SEEK_CUR);
}
```

输出:

```
0x202, 1
0x201, 1
0x202, 2
0x201, 2
0x202, 3
0x201, 3
0x201, 3
0x202, 4
0x201, 4
0x201, 4
0x201, 4
0x202, 5
0x201, 5
0x201, 5
0x201, 5
0x201, 5
0x201, 5
0x201, 5
0x201, 5
...
```

这里打印了type和id,type 0x202代表了RES\_TABLE\_TYPE\_SPEC\_TYPE，0x201代表了RES\_TABLE\_TYPE\_TYPE。

而id则代表了资源类型字符串池里面的顺序,例如1是anim，2是attr，3是bool...

翻译过来就是:

```
RES_TABLE_TYPE_SPEC_TYPE,  anim
RES_TABLE_TYPE_TYPE,       anim
RES_TABLE_TYPE_SPEC_TYPE,  attr
RES_TABLE_TYPE_TYPE,       attr
RES_TABLE_TYPE_SPEC_TYPE,  bool
RES_TABLE_TYPE_TYPE,       bool
RES_TABLE_TYPE_TYPE,       bool
RES_TABLE_TYPE_SPEC_TYPE,  color
RES_TABLE_TYPE_TYPE,       color
RES_TABLE_TYPE_TYPE,       color
RES_TABLE_TYPE_TYPE,       color
RES_TABLE_TYPE_SPEC_TYPE,  dimen
RES_TABLE_TYPE_TYPE,       dimen
RES_TABLE_TYPE_TYPE,       dimen
RES_TABLE_TYPE_TYPE,       dimen
RES_TABLE_TYPE_TYPE,       dimen
RES_TABLE_TYPE_TYPE,       dimen
RES_TABLE_TYPE_TYPE,       dimen
RES_TABLE_TYPE_TYPE,       dimen
...
```

好的,因为信息量已经有点大了,本节先到这里。接下来是整个resources.arsc中最重要的资源的具体定义会在下一篇笔记中介绍,完整的demo代码也会在下一篇文章给出。