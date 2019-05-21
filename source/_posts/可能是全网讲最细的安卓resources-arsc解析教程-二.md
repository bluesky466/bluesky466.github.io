title: 可能是全网讲最细的安卓resources.arsc解析教程(二)
date: 2019-05-21 19:19:09
tags:
    - 技术相关
    - Android
---

上篇[博客](http://blog.islinjw.cn/2019/05/18/%E5%8F%AF%E8%83%BD%E6%98%AF%E5%85%A8%E7%BD%91%E8%AE%B2%E6%9C%80%E7%BB%86%E7%9A%84%E5%AE%89%E5%8D%93resources-arsc%E8%A7%A3%E6%9E%90%E6%95%99%E7%A8%8B-%E4%B8%80/)写到,Package资源剩下的部分是由多组RES\_TABLE\_TYPE\_SPEC\_TYPE和RES\_TABLE\_TYPE\_TYPE构成的。

一个RES\_TABLE\_TYPE\_SPEC\_TYPE后面跟着一个或者多个RES\_TABLE\_TYPE\_TYPE构成一种类型的资源的描述(例如string类型、bool类型、dimen类型等)

# RES\_TABLE\_TYPE\_SPEC\_TYPE

我们接着来看看RES\_TABLE\_TYPE\_SPEC\_TYPE的头部结构体:

```
/**
 * A specification of the resources defined by a particular type.
 *
 * There should be one of these chunks for each resource type.
 *
 * This structure is followed by an array of integers providing the set of
 * configuration change flags (ResTable_config::CONFIG_*) that have multiple
 * resources for that configuration.  In addition, the high bit is set if that
 * resource has been made public.
 */
struct ResTable_typeSpec
{
    struct ResChunk_header header;

    // The type identifier this chunk is holding.  Type IDs start
    // at 1 (corresponding to the value of the type bits in a
    // resource identifier).  0 is invalid.
    uint8_t id;

    // Must be 0.
    uint8_t res0;
    // Must be 0.
    uint16_t res1;

    // Number of uint32_t entry configuration masks that follow.
    uint32_t entryCount;

    enum {
        // Additional flag indicating an entry is public.
        SPEC_PUBLIC = 0x40000000
    };
};
```


从注释中可以知道ResTable\_typeSpec头部后面会跟着entryCount个uint32\_t,代表这种类型有entryCount个数据,并且每个uint32\_t标识了这个数据在哪些configuration下有特殊的值。

这些configuration可能是不同的地区、不同的屏幕分辨率、不同的sdk版本等:

```
// Flags indicating a set of config values.  These flag constants must
// match the corresponding ones in android.content.pm.ActivityInfo and
// attrs_manifest.xml.
enum {
    CONFIG_MCC = ACONFIGURATION_MCC,
    CONFIG_MNC = ACONFIGURATION_MCC,
    CONFIG_LOCALE = ACONFIGURATION_LOCALE,
    CONFIG_TOUCHSCREEN = ACONFIGURATION_TOUCHSCREEN,
    CONFIG_KEYBOARD = ACONFIGURATION_KEYBOARD,
    CONFIG_KEYBOARD_HIDDEN = ACONFIGURATION_KEYBOARD_HIDDEN,
    CONFIG_NAVIGATION = ACONFIGURATION_NAVIGATION,
    CONFIG_ORIENTATION = ACONFIGURATION_ORIENTATION,
    CONFIG_DENSITY = ACONFIGURATION_DENSITY,
    CONFIG_SCREEN_SIZE = ACONFIGURATION_SCREEN_SIZE,
    CONFIG_SMALLEST_SCREEN_SIZE = ACONFIGURATION_SMALLEST_SCREEN_SIZE,
    CONFIG_VERSION = ACONFIGURATION_VERSION,
    CONFIG_SCREEN_LAYOUT = ACONFIGURATION_SCREEN_LAYOUT,
    CONFIG_UI_MODE = ACONFIGURATION_UI_MODE,
    CONFIG_LAYOUTDIR = ACONFIGURATION_LAYOUTDIR,
};
```

这里直接举个例子,例如我们可能会在res/values目录下创建一些bool配置:

```
<bool name="abc_action_bar_embed_tabs">true</bool>
<bool name="abc_allow_stacked_button_bar">false</bool>
<bool name="abc_config_actionMenuItemAllCaps">true</bool>
```

然后可能在竖屏的情况下我们不需要显示action bar,所以在res/values-port目录下我们会把abc\_action\_bar\_embed\_tabs的值设置成false

```
<bool name="abc_action_bar_embed_tabs">false</bool>
```

然后下面代码就能在横屏、竖屏下拿到不同的配置了:

```
context.getResources().getBoolean(R.bool.abc_action_bar_embed_tabs);
```

在代码里面,我们可以先读取ResTable\_typeSpec,然后根据entryCount得到这种类型有多少个数据(例如这里的bool就有abc\_action\_bar\_embed\_tabs、abc\_allow\_stacked\_button\_bar、abc\_config\_actionMenuItemAllCaps三个数据,所以bool类型下的entryCount就是3),然后继续读entryCount个uint32\_t,读出来就是每个数据在哪些configuration下有特殊的值。

```
//printStringFromStringsPool:

void printStringFromStringsPool(uint32_t* pOffsets, char* pStringsStart, uint32_t stringIndex, uint32_t isUtf8) {
    //前面两个字节是长度,要跳过
    char* str = pStringsStart + *(pOffsets + stringIndex) + 2;
    if(isUtf8) {
        printf("%s\n", str);
    } else {
        printUtf16String((char16_t*)str);
    }
}


//main:

...

ResTable_typeSpec typeSpecHeader;
uint32_t config;
uint16_t type;
while(fread((void*)&type, sizeof(u_int16_t), 1, pFile) != 0) {
    fseek(pFile, -sizeof(uint16_t), SEEK_CUR);
    if(RES_TABLE_TYPE_SPEC_TYPE == type) {
        fread((void*)&typeSpecHeader, sizeof(struct ResTable_typeSpec), 1, pFile);
        printf("type: id=0x%x,name=", typeSpecHeader.id);
        printStringFromStringsPool(
                (uint32_t*)pTypeStrings,
                (char*)pTypeStrings + typeStringPoolHeader.stringsStart - sizeof(struct ResStringPool_header),
                typeSpecHeader.id - 1,
                typeStringPoolHeader.flags & ResStringPool_header::UTF8_FLAG
        );

        for(int i = 0 ; i < typeSpecHeader.entryCount ; i++) {
            fread((void*)&config, sizeof(uint32_t), 1, pFile);
            printf("%x\n",config);
        }
    } 
	...
}

...
```

我们直接找到bool类型下的打印:

```
...
type:id=3,name=bool
80
0
0

...
```

可以看到bool类型下的确有三个uint32\_t,分别是80、0、0。这个80代表的就是CONFIG\_ORIENTATION,也就是说这个数据在不同的屏幕方向下面会有和默认值不同的值。而0则代表了这个数据只有一个默认值,不会跟着configuration的变化而改变:

```
//configuration.h
ACONFIGURATION_ORIENTATION = 0x0080,

//ResTable_config里面的enum
CONFIG_ORIENTATION = ACONFIGURATION_ORIENTATION,

```

让我们返回去对比下:

```
// res/values目录下
<bool name="abc_action_bar_embed_tabs">true</bool>
<bool name="abc_allow_stacked_button_bar">false</bool>
<bool name="abc_config_actionMenuItemAllCaps">true</bool>

// res/values-port目录下
<bool name="abc_action_bar_embed_tabs">false</bool>
```

第一个abc\_action\_bar\_embed\_tabs在不同的屏幕方向下可能值会改变,所以它的uint32\_t值是80,也就是CONFIG\_ORIENTATION,而abc\_allow\_stacked\_button_bar 和abc\_config\_actionMenuItemAllCaps只有默认的配置,所以他们的uint32\_t都是0。

所以RES\_TABLE\_TYPE\_SPEC\_TYPE的作用就是将数据受到哪些configuration影响都标识出来。

在读取数据的时候先看看它是否会受configuration影响,如果不会,直接读默认的RES\_TABLE\_TYPE\_TYPE里面的默认值就好,否则就根据当前的configuration去到后面对应的RES\_TABLE\_TYPE\_TYPE下面读取对应的值了。

# RES\_TABLE\_TYPE\_TYPE

讲的这里终于到了最重要的部分,我们在xml里面配的值,都会在RES\_TABLE\_TYPE\_TYPE里面体现出来。

我们照例先来看看它的头部结构体:

```
/**
 * A collection of resource entries for a particular resource data
 * type. Followed by an array of uint32_t defining the resource
 * values, corresponding to the array of type strings in the
 * ResTable_package::typeStrings string block. Each of these hold an
 * index from entriesStart; a value of NO_ENTRY means that entry is
 * not defined.
 *
 * There may be multiple of these chunks for a particular resource type,
 * supply different configuration variations for the resource values of
 * that type.
 *
 * It would be nice to have an additional ordered index of entries, so
 * we can do a binary search if trying to find a resource by string name.
 */
struct ResTable_type
{
    struct ResChunk_header header;

    enum {
        NO_ENTRY = 0xFFFFFFFF
    };

    // The type identifier this chunk is holding.  Type IDs start
    // at 1 (corresponding to the value of the type bits in a
    // resource identifier).  0 is invalid.
    uint8_t id;

    // Must be 0.
    uint8_t res0;
    // Must be 0.
    uint16_t res1;

    // Number of uint32_t entry indices that follow.
    uint32_t entryCount;

    // Offset from header where ResTable_entry data starts.
    uint32_t entriesStart;

    ResTable_config config;
};
```

这个ResTable\_type里有个config成员,它就是具体的配置了,我们可以把它打印出来:

```
else if(RES_TABLE_TYPE_TYPE == type) {
    fread((void*)&typeHeader, sizeof(struct ResTable_type), 1, pFile);
    printConfig(typeHeader.config);
    ...
}
```

找到bool的那一段,可以看到它有两个RES\_TABLE\_TYPE\_TYPE,第一个是默认的配置(values目录),第二个是port下的配置(values-port目录):

```
...
type: id=0x3,name=bool
80
0
0
config : 
config : port
...
```

然后根据注释的说明我们知道,ResTable\_type头部后跟着entryCount个uint32\_t,代表了每个entry相对entriesStart的偏移。这里和RES\_STRING\_POOL\_TYPE有点像,也是从偏移数组读取数据的偏移值,然后从entriesStart进行偏移得到数据的地址。

{% img /可能是全网讲最细的安卓resources_arsc解析教程二/1.png %}


那entriesStart后面的entry是什么呢？其实entry有两种类型ResTable\_entry和ResTable\_map\_entry。

他们其实是有继承关系的,ResTable\_map\_entry是ResTable\_entry的子类(这里的继承关系是c++里面的继承关系,前面我们都是用c语言去讲的,但是这里必须引入c++了,不过也是最基础的继承而已,大家可以自行搜索下)。


```
/**
 * This is the beginning of information about an entry in the resource
 * table.  It holds the reference to the name of this entry, and is
 * immediately followed by one of:
 *   * A Res_value structure, if FLAG_COMPLEX is -not- set.
 *   * An array of ResTable_map structures, if FLAG_COMPLEX is set.
 *     These supply a set of name/value mappings of data.
 */
struct ResTable_entry
{
    // Number of bytes in this structure.
    uint16_t size;

    enum {
        // If set, this is a complex entry, holding a set of name/value
        // mappings.  It is followed by an array of ResTable_map structures.
        FLAG_COMPLEX = 0x0001,
        // If set, this resource has been declared public, so libraries
        // are allowed to reference it.
        FLAG_PUBLIC = 0x0002,
        // If set, this is a weak resource and may be overriden by strong
        // resources of the same name/type. This is only useful during
        // linking with other resource tables.
        FLAG_WEAK = 0x0004
    };
    uint16_t flags;

    // Reference into ResTable_package::keyStrings identifying this entry.
    struct ResStringPool_ref key;
};


/**
 * Extended form of a ResTable_entry for map entries, defining a parent map
 * resource from which to inherit values.
 */
struct ResTable_map_entry : public ResTable_entry
{
    // Resource identifier of the parent mapping, or 0 if there is none.
    ResTable_ref parent;
    // Number of name/value pairs that follow for FLAG_COMPLEX.
    uint32_t count;
};
```

看到注释我们可以知道, ResTable\_entry有个flags成员变量,如果它的FLAG\_COMPLEX位被置1(也就是说flags & 0x0001 != 0),则它是个ResTable\_map\_entry结构。

两种结构的不同之处在于ResTable\_entry后面跟着的是一个Res\_value,而ResTable\_map\_entry后面跟着的是多个name/value键值对,这个键值对是用struct ResTable\_map来表示的。


## ResTable\_entry

我们先从ResTable\_entry讲起,我们读完struct ResTable\_type头部信息之后继续将offset数组和entriesStart开始到剩下的部分都读进去保存到pOffset和pData中。


接着就可以用*(pOffsets + i)得到每个entry的偏移,再与entriesStart相加得到entry的具体位置。这里有一点需要注意的是如果offset是ResTable_type::NO\_ENTRY,也就是0xFFFFFFFF的时候,代表它是无效的,直接跳过即可:


```
else if(RES_TABLE_TYPE_TYPE == type) {
    fread((void*)&typeHeader, sizeof(struct ResTable_type), 1, pFile);
    printConfig(typeHeader.config);

    // 实际struct ResTable_type的大小可能不同sdk版本不一样,所以typeHeader.header.headerSize才是真正的头部大小
    fseek(pFile, typeHeader.header.headerSize -  sizeof(struct ResTable_type), SEEK_CUR);;

    uint32_t* pOffsets = (uint32_t*)malloc(typeHeader.entryCount * sizeof(uint32_t));
    fread((void*)pOffsets, sizeof(uint32_t), typeHeader.entryCount, pFile);

    unsigned char* pData = (unsigned char*)malloc(typeHeader.header.size - typeHeader.entriesStart);
    fread((void*)pData, typeHeader.header.size - typeHeader.entriesStart, 1, pFile);

    for(int i = 0 ; i< typeHeader.entryCount ; i++) {
        uint32_t offset = *(pOffsets + i);
        if(offset == ResTable_type::NO_ENTRY) {
            continue;
        }
        struct ResTable_entry* pEntry = (struct ResTable_entry*)(pData + offset);
        printf("entryIndex: 0x%x, key :\n", i);
        printStringFromStringsPool(
            (uint32_t*)pKeyStrings,
            (char*)pKeyStrings + keyStringPoolHeader.stringsStart - sizeof(struct ResStringPool_header),
            pEntry->key.index,
            keyStringPoolHeader.flags & ResStringPool_header::UTF8_FLAG
        );
        if(pEntry->flags & ResTable_entry::FLAG_COMPLEX) {
           ...
        } else {
            struct Res_value* pValue = (struct Res_value*)((unsigned char*)pEntry + sizeof(struct ResTable_entry));
            printf("value :\n");
            printValue(pValue, globalStringPoolHeader, pGlobalStrings);
            printf("\n");
        }
    }
    free(pOffsets);
    free(pData);
}
```

pEntry->key.index就是资源的key在资源key字符串池中的序号了,直接打印即可。

然后找到struct ResTable\_entry后面跟着的struct Res\_value,这个结构体里面就是资源的值。但是这个值的获取比较复杂,我们先来看看这个结构体的定义:

```

/**
 * Representation of a value in a resource, supplying type
 * information.
 */
struct Res_value
{
    // Number of bytes in this structure.
    uint16_t size;

    // Always set to 0.
    uint8_t res0;

    // Type of the data value.
    enum {
        // The 'data' is either 0 or 1, specifying this resource is either
        // undefined or empty, respectively.
        TYPE_NULL = 0x00,
        // The 'data' holds a ResTable_ref, a reference to another resource
        // table entry.
        TYPE_REFERENCE = 0x01,
        // The 'data' holds an attribute resource identifier.
        TYPE_ATTRIBUTE = 0x02,
        // The 'data' holds an index into the containing resource table's
        // global value string pool.
        TYPE_STRING = 0x03,
        // The 'data' holds a single-precision floating point number.
        TYPE_FLOAT = 0x04,
        // The 'data' holds a complex number encoding a dimension value,
        // such as "100in".
        TYPE_DIMENSION = 0x05,
        // The 'data' holds a complex number encoding a fraction of a
        // container.
        TYPE_FRACTION = 0x06,
        // The 'data' holds a dynamic ResTable_ref, which needs to be
        // resolved before it can be used like a TYPE_REFERENCE.
        TYPE_DYNAMIC_REFERENCE = 0x07,
        // The 'data' holds an attribute resource identifier, which needs to be resolved
        // before it can be used like a TYPE_ATTRIBUTE.
        TYPE_DYNAMIC_ATTRIBUTE = 0x08,
        
        // Beginning of integer flavors...
        TYPE_FIRST_INT = 0x10,

        // The 'data' is a raw integer value of the form n..n.
        TYPE_INT_DEC = 0x10,
        // The 'data' is a raw integer value of the form 0xn..n.
        TYPE_INT_HEX = 0x11,
        // The 'data' is either 0 or 1, for input "false" or "true" respectively.
        TYPE_INT_BOOLEAN = 0x12,

        // Beginning of color integer flavors...
        TYPE_FIRST_COLOR_INT = 0x1c,

        // The 'data' is a raw integer value of the form #aarrggbb.
        TYPE_INT_COLOR_ARGB8 = 0x1c,
        // The 'data' is a raw integer value of the form #rrggbb.
        TYPE_INT_COLOR_RGB8 = 0x1d,
        // The 'data' is a raw integer value of the form #argb.
        TYPE_INT_COLOR_ARGB4 = 0x1e,
        // The 'data' is a raw integer value of the form #rgb.
        TYPE_INT_COLOR_RGB4 = 0x1f,

        // ...end of integer flavors.
        TYPE_LAST_COLOR_INT = 0x1f,

        // ...end of integer flavors.
        TYPE_LAST_INT = 0x1f
    };
    uint8_t dataType;
    
     // Structure of complex data values (TYPE_UNIT and TYPE_FRACTION)
    enum {
        // Where the unit type information is.  This gives us 16 possible
        // types, as defined below.
        COMPLEX_UNIT_SHIFT = 0,
        COMPLEX_UNIT_MASK = 0xf,

        // TYPE_DIMENSION: Value is raw pixels.
        COMPLEX_UNIT_PX = 0,
        // TYPE_DIMENSION: Value is Device Independent Pixels.
        COMPLEX_UNIT_DIP = 1,
        // TYPE_DIMENSION: Value is a Scaled device independent Pixels.
        COMPLEX_UNIT_SP = 2,
        // TYPE_DIMENSION: Value is in points.
        COMPLEX_UNIT_PT = 3,
        // TYPE_DIMENSION: Value is in inches.
        COMPLEX_UNIT_IN = 4,
        // TYPE_DIMENSION: Value is in millimeters.
        COMPLEX_UNIT_MM = 5,

        // TYPE_FRACTION: A basic fraction of the overall size.
        COMPLEX_UNIT_FRACTION = 0,
        // TYPE_FRACTION: A fraction of the parent size.
        COMPLEX_UNIT_FRACTION_PARENT = 1,

        // Where the radix information is, telling where the decimal place
        // appears in the mantissa.  This give us 4 possible fixed point
        // representations as defined below.
        COMPLEX_RADIX_SHIFT = 4,
        COMPLEX_RADIX_MASK = 0x3,

        // The mantissa is an integral number -- i.e., 0xnnnnnn.0
        COMPLEX_RADIX_23p0 = 0,
        // The mantissa magnitude is 16 bits -- i.e, 0xnnnn.nn
        COMPLEX_RADIX_16p7 = 1,
        // The mantissa magnitude is 8 bits -- i.e, 0xnn.nnnn
        COMPLEX_RADIX_8p15 = 2,
        // The mantissa magnitude is 0 bits -- i.e, 0x0.nnnnnn
        COMPLEX_RADIX_0p23 = 3,
        
        // Where the actual value is.  This gives us 23 bits of
        // precision.  The top bit is the sign.
        COMPLEX_MANTISSA_SHIFT = 8,
        COMPLEX_MANTISSA_MASK = 0xffffff
    };

    // Possible data values for TYPE_NULL.
    enum {
        // The value is not defined.
        DATA_NULL_UNDEFINED = 0,
        // The value is explicitly defined as empty.
        DATA_NULL_EMPTY = 1
    };

    // The data for this item, as interpreted according to dataType.
    typedef uint32_t data_type;
    data_type data;
};
```

我们先需要根据dataType判断这个值是什么类型的,然后再根据不同的类型,从data读取具体的值。读取的方法比较复杂,我就不具体讲解,大家可以参考我的demo代码理解。

我们找到bool部分的打印,可以看到key和value就都打印出来了:

```
type: id=0x3,name=bool
80
0
0
config :
entryIndex: 0x0, key :
abc_action_bar_embed_tabs
value :
(boolean) true

entryIndex: 0x1, key :
abc_allow_stacked_button_bar
value :
(boolean) false

entryIndex: 0x2, key :
abc_config_actionMenuItemAllCaps
value :
(boolean) true

config : port
entryIndex: 0x0, key :
abc_action_bar_embed_tabs
value :
(boolean) false
```

## ResTable\_map\_entry

从上面可以看出来ResTable\_entry代表的是普通键值对的资源如string、bool、drawable等,那ResTable\_map\_entry又代表的是啥呢?

其实它代表的是类型style、attr的资源:

```
<attr name="buttonTintMode">
	<enum name="src_over" value="3"/>
	<enum name="src_in" value="5"/>
	<enum name="src_atop" value="9"/>
	<enum name="multiply" value="14"/>
	<enum name="screen" value="15"/>
	<enum name="add" value="16"/>
</attr>

<style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
</style>
```

像上面的R.attr.buttonTintMode和R.style.AppTheme的值都需要用一个map去表示。

我们来看看struct ResTable\_map\_entry:

```
/**
 * Extended form of a ResTable_entry for map entries, defining a parent map
 * resource from which to inherit values.
 */
struct ResTable_map_entry : public ResTable_entry
{
    // Resource identifier of the parent mapping, or 0 if there is none.
    ResTable_ref parent;
    // Number of name/value pairs that follow for FLAG_COMPLEX.
    uint32_t count;
};
```

它的parent成员变量就定义了这个style的parent,count成员变量则代表了这个map的大小,也就是ResTable\_map\_entry后面跟着的键值对的数量。

### 资源的id

struct ResTable\_ref也是一个需要重点讲解的结构体,它的定义很简单:

```
/**
 *  This is a reference to a unique entry (a ResTable_entry structure)
 *  in a resource table.  The value is structured as: 0xpptteeee,
 *  where pp is the package index, tt is the type index in that
 *  package, and eeee is the entry index in that type.  The package
 *  and type values start at 1 for the first item, to help catch cases
 *  where they have not been supplied.
 */
struct ResTable_ref
{
    uint32_t ident;
};
```

这个ident代表的就是资源的id。这个值其实我们在java里面也能看到:

```
public final class R {
	...
	public static final class bool {
	    public static final int abc_action_bar_embed_tabs=0x7f030000;
	    public static final int abc_allow_stacked_button_bar=0x7f030001;
	    public static final int abc_config_actionMenuItemAllCaps=0x7f030002;
	  }
	...
}
```

资源的id其实是有固定的格式和含义的,它的格式如下:

> 0xpptteeee

头一个字节保存了packageId,接着的一个字节保存了typeId,后面的两个字节保存了entryIndex。例如我们的abc\_allow\_stacked\_button\_bar=0x7f030001,它的packageId=0x7f, typeId=0x3, entryIndex=0x1。

我们在解析package资源的时候就已经把package id打印了出来,它就是0x7f:

```
type:512, headSize:288, size:188068, id:7f, packageName:com.cvte.tv.myapplication
```

而在后面解析资源的时候也把typeId和entryIndex打印了出来:

```
type: id=0x3,name=bool
80
0
0
config :
entryIndex: 0x0, key :
abc_action_bar_embed_tabs
value :
(boolean) true

entryIndex: 0x1, key :
abc_allow_stacked_button_bar
```

于是乎我们就能定位到abc\_allow\_stacked\_button\_bar这个资源了。

所以我们的style的parent.ident就可以定位到style的parent资源。

有时候我们会看到packageId是0x01,在我们的resource.arsc里面找不到对应的package。这个package指定其实是系统资源包,我们在xml里面配置的@android:color/black就会使用到系统资源包里面的资源,这个资源是不会打包进我们的应用的:

```
...

type: id=0x4,name=color

...

entryIndex: 0x41, key :
primary_dark_material_dark
value :
(reference) 0x0106000c

...
```

### ResTable\_map

ResTable\_map\_entry后面跟着的键值对数组其实就是一个个的ResTable\_map:

{% img /可能是全网讲最细的安卓resources_arsc解析教程二/2.png %}

struct ResTable\_map定义如下:

```
/**
 * A single name/value mapping that is part of a complex resource
 * entry.
 */
struct ResTable_map
{
    // The resource identifier defining this mapping's name.  For attribute
    // resources, 'name' can be one of the following special resource types
    // to supply meta-data about the attribute; for all other resource types
    // it must be an attribute resource.
    ResTable_ref name;

    // Special values for 'name' when defining attribute resources.
    enum {
        // This entry holds the attribute's type code.
        ATTR_TYPE = Res_MAKEINTERNAL(0),

        // For integral attributes, this is the minimum value it can hold.
        ATTR_MIN = Res_MAKEINTERNAL(1),

        // For integral attributes, this is the maximum value it can hold.
        ATTR_MAX = Res_MAKEINTERNAL(2),

        // Localization of this resource is can be encouraged or required with
        // an aapt flag if this is set
        ATTR_L10N = Res_MAKEINTERNAL(3),

        // for plural support, see android.content.res.PluralRules#attrForQuantity(int)
        ATTR_OTHER = Res_MAKEINTERNAL(4),
        ATTR_ZERO = Res_MAKEINTERNAL(5),
        ATTR_ONE = Res_MAKEINTERNAL(6),
        ATTR_TWO = Res_MAKEINTERNAL(7),
        ATTR_FEW = Res_MAKEINTERNAL(8),
        ATTR_MANY = Res_MAKEINTERNAL(9)

    };
    
     // Bit mask of allowed types, for use with ATTR_TYPE.
    enum {
        // No type has been defined for this attribute, use generic
        // type handling.  The low 16 bits are for types that can be
        // handled generically; the upper 16 require additional information
        // in the bag so can not be handled generically for TYPE_ANY.
        TYPE_ANY = 0x0000FFFF,

        // Attribute holds a references to another resource.
        TYPE_REFERENCE = 1<<0,

        // Attribute holds a generic string.
        TYPE_STRING = 1<<1,

        // Attribute holds an integer value.  ATTR_MIN and ATTR_MIN can
        // optionally specify a constrained range of possible integer values.
        TYPE_INTEGER = 1<<2,

        // Attribute holds a boolean integer.
        TYPE_BOOLEAN = 1<<3,

        // Attribute holds a color value.
        TYPE_COLOR = 1<<4,

        // Attribute holds a floating point value.
        TYPE_FLOAT = 1<<5,

        // Attribute holds a dimension value, such as "20px".
        TYPE_DIMENSION = 1<<6,

        // Attribute holds a fraction value, such as "20%".
        TYPE_FRACTION = 1<<7,

        // Attribute holds an enumeration.  The enumeration values are
        // supplied as additional entries in the map.
        TYPE_ENUM = 1<<16,

        // Attribute holds a bitmaks of flags.  The flag bit values are
        // supplied as additional entries in the map.
        TYPE_FLAGS = 1<<17
        };

    // Enum of localization modes, for use with ATTR_L10N.
    enum {
        L10N_NOT_REQUIRED = 0,
        L10N_SUGGESTED    = 1
    };

    // This mapping's value.
    Res_value value;
};
```

它的name代表的就是这个键值对的key,而它的value代表的就是键值对的值。

name同样的是个struct ResTable\_ref,它同样可以从资源id拿到对应的资源,但是这个name有点特殊,如果是它的ident的值是下面枚举中的一个的话:

```
#define Res_MAKEINTERNAL(entry) (0x01000000 | (entry&0xFFFF))

enum {
    // This entry holds the attribute's type code.
    ATTR_TYPE = Res_MAKEINTERNAL(0),

    // For integral attributes, this is the minimum value it can hold.
    ATTR_MIN = Res_MAKEINTERNAL(1),

    // For integral attributes, this is the maximum value it can hold.
    ATTR_MAX = Res_MAKEINTERNAL(2),

    // Localization of this resource is can be encouraged or required with
    // an aapt flag if this is set
    ATTR_L10N = Res_MAKEINTERNAL(3),

    // for plural support, see android.content.res.PluralRules#attrForQuantity(int)
    ATTR_OTHER = Res_MAKEINTERNAL(4),
    ATTR_ZERO = Res_MAKEINTERNAL(5),
    ATTR_ONE = Res_MAKEINTERNAL(6),
    ATTR_TWO = Res_MAKEINTERNAL(7),
    ATTR_FEW = Res_MAKEINTERNAL(8),
    ATTR_MANY = Res_MAKEINTERNAL(9)
};
```

例如如果index==0x01000000,就代表name是ATTR\_TYPE,也代表这个资源是attr。

此时,它的value也是特殊的,是下面枚举中的一个,代表attr的类型:

```
enum {
    // No type has been defined for this attribute, use generic
    // type handling.  The low 16 bits are for types that can be
    // handled generically; the upper 16 require additional information
    // in the bag so can not be handled generically for TYPE_ANY.
    TYPE_ANY = 0x0000FFFF,

    // Attribute holds a references to another resource.
    TYPE_REFERENCE = 1<<0,

    // Attribute holds a generic string.
    TYPE_STRING = 1<<1,

    // Attribute holds an integer value.  ATTR_MIN and ATTR_MIN can
    // optionally specify a constrained range of possible integer values.
    TYPE_INTEGER = 1<<2,

    // Attribute holds a boolean integer.
    TYPE_BOOLEAN = 1<<3,

    // Attribute holds a color value.
    TYPE_COLOR = 1<<4,

    // Attribute holds a floating point value.
    TYPE_FLOAT = 1<<5,

    // Attribute holds a dimension value, such as "20px".
    TYPE_DIMENSION = 1<<6,

    // Attribute holds a fraction value, such as "20%".
    TYPE_FRACTION = 1<<7,

    // Attribute holds an enumeration.  The enumeration values are
    // supplied as additional entries in the map.
    TYPE_ENUM = 1<<16,

    // Attribute holds a bitmaks of flags.  The flag bit values are
    // supplied as additional entries in the map.
    TYPE_FLAGS = 1<<17
};
```

解析代码如下:

```
if(pEntry->flags & ResTable_entry::FLAG_COMPLEX) {
    struct ResTable_map_entry* pMapEntry = (struct ResTable_map_entry*)(pData + offset);
    for(int i = 0; i <pMapEntry->count ; i++) {
        struct ResTable_map* pMap = (struct ResTable_map*)(pData + offset + pMapEntry->size + i * sizeof(struct ResTable_map_entry));
        printf("\tname:0x%x, valueType:%u, value:%u\n", pMap->name.ident, pMap->value.dataType, pMap->value.data);
    }
}
```

让我们找到buttonTintMode的打印


```
entryIndex: 0x69, key :
buttonTintMode
    name:0x1000000, valueType:16, value:65536
    name:0x7f070019, valueType:16, value:16
    name:0x7f070050, valueType:16, value:14
    name:0x7f070061, valueType:16, value:15
    name:0x7f070078, valueType:16, value:9
    name:0x7f070079, valueType:16, value:5
    name:0x7f07007a, valueType:16, value:3
```

第一个ResTable\_ref的name的indent的值是0x1000000,就代表name是ATTR\_TYPE,也代表这个资源是attr。然后value是65536,也就是TYPE\_ENUM。


然后我们顺便找下7f070019、7f070050、7f070061、7f070078、7f070079、7f07007a资源的定义:

```
...

type: id=0x7,name=id

...

entryIndex: 0x19, key :
add
value :
(boolean) false

...

entryIndex: 0x50, key :
multiply
value :
(boolean) false

...

entryIndex: 0x61, key :
screen
value :
(boolean) false

...

entryIndex: 0x78, key :
src_atop
value :
(boolean) false

entryIndex: 0x79, key :
src_in
value :
(boolean) false

entryIndex: 0x7a, key :
src_over
value :
(boolean) false
```

# Demo

完整的demo可以在github上找到:

https://github.com/bluesky466/ResourcesArscDemo

呼~长舒一口气,终于大功告成。