title: 安卓UVC控制协议入门
date: 2024-09-20 00:40:34
tags:
    - 技术相关
    - Android
---


最近的项目里面需要对UVC摄像头进行操控,简单的了解了下相关的知识。

首先UVC全称为USB video(device) class,是微软与另外几家设备厂商联合推出的为USB视频捕获设备定义的协议标准，目前已成为USB org标准之一。在[USB中文网](https://www.usbzh.com/article/detail-80.html)上其实有比较详细的描述,但是新手直接上来就看这个协议其实是比较懵逼的。所以可以参考[UVCCamera](https://github.com/saki4510t/UVCCamera)这个安卓项目的源码去辅助理解。

# USB描述符

当我们连接到一个UVC设备之后其实第一步应该先获取它的描述符来看看它具体支持哪些操作。UVCCamera里是使用[libusb](https://github.com/libusb/libusb)获取到USB的设备描述符之后在[uvc_scan_control](https://github.com/saki4510t/UVCCamera/blob/master/libuvccamera/src/main/jni/libuvc/src/device.c#L904)里面解析的。实际上我们也可以利用安卓应用层的UsbDeviceConnection.getRawDescriptors接口获取到USB的描述符然后再java层解析:

```kotlin
val manager = context.getSystemService(AppCompatActivity.USB_SERVICE) as UsbManager
val deviceIterator: Iterator<UsbDevice> = manager.deviceList.values.iterator()
while (deviceIterator.hasNext()) {
    val device = deviceIterator.next()
    if (device.vendorId == targetVid && device.productId == targetPid) {
        val connect = manager.openDevice(device)
        val desc = connect.rawDescriptors
        // 解析usb描述符
        connect.close()
        return
    }
}
```

这里拿到的是一个byte数组,我们先要理解什么是usb描述符才能去解析它。usb描述符其实就是描述usb的属性和用途的,四种主要描述符的逻辑结构大概如下:

{% img /AndroidUvcControl/2.png %}

- 设备描述符: 每一个USB设备只有一个设备描述符，主要向主机说明设备类型、端点0最大包长、设备版本、配置数量等等
- 配置描述符: 每一个USB设备至少有一个或者多个配置描述符，但是主机同一时间只能选择某一种配置，标准配置描述符主要向主机描述当前配置下的设备属性、所需电流、支持的接口数、配置描述符集合长度等等。
- 接口描述符 : 每一个USB配置下至少有一个或者多个接口描述符，接口描述符主要说明设备类型、此接口下使用的端点数（不包括0号端点），一个接口就是实现一种功能，实现这种功能可能需要端点0就够了，可能还需要其它的端点配合。
- 端点描述符: 每一个USB接口下可以有多个端点描述符，端点描述符用来描述符端点的各种属性。

端点是实现USB设备功能的物理缓冲区实体，USB主机和设备是通过端点进行数据交互的

## 描述符解析

所有类型的描述符的前两个字节定义都是一样的,第一个字节指定描述符的长度，而第二个字节表示描述符类型。所以我们拿到rawDescriptors之后可以用下面的代码去遍历解析:

```kotlin
var index = 0
val descriptorTypes = mapOf(
    0x01.toByte() to "DEVICE",
    0x02.toByte() to "CONFIG",
    0x04.toByte() to "INTERFACE",
    0x05.toByte() to "ENDPOINT",
)

while (index < desc.size) {
    descriptorTypes[desc[index + 1]]?.let {
        val indent = " ".repeat(desc[index + 1].toInt())
        Log.d(TAG, "${indent}$it")
    }
    index += desc[index]
}
```

我这个调试设备的[描述符](https://blog.islinjw.cn/AndroidUvcControl/desc.bin)解析如下:

```
 DEVICE
  CONFIG
    INTERFACE
     ENDPOINT
    INTERFACE
     ENDPOINT
    INTERFACE
     ENDPOINT
     ENDPOINT
```

可以看到它有一个设备描述符,这个设备描述符下有个一个配置描述符,这个配置描述符下有三个接口描述符,每个接口描述符下又有一到两个端点描述符。

知道了描述符的类型就能在USB标准里找到它具体的数据结构去解析,例如[设备描述符](https://github.com/libusb/libusb/blob/467b6a8896daea3d104958bf0887312c5d14d150/libusb/libusb.h#L578)的定义如下:

```c
struct libusb_device_descriptor {
	/** Size of this descriptor (in bytes) */
	uint8_t  bLength;

	/** Descriptor type. Will have value
	 * \ref libusb_descriptor_type::LIBUSB_DT_DEVICE LIBUSB_DT_DEVICE in this
	 * context. */
	uint8_t  bDescriptorType;

	/** USB specification release number in binary-coded decimal. A value of
	 * 0x0200 indicates USB 2.0, 0x0110 indicates USB 1.1, etc. */
	uint16_t bcdUSB;

	/** USB-IF class code for the device. See \ref libusb_class_code. */
	uint8_t  bDeviceClass;

	/** USB-IF subclass code for the device, qualified by the bDeviceClass
	 * value */
	uint8_t  bDeviceSubClass;

	/** USB-IF protocol code for the device, qualified by the bDeviceClass and
	 * bDeviceSubClass values */
	uint8_t  bDeviceProtocol;

	/** Maximum packet size for endpoint 0 */
	uint8_t  bMaxPacketSize0;

	/** USB-IF vendor ID */
	uint16_t idVendor;

	/** USB-IF product ID */
	uint16_t idProduct;

	/** Device release number in binary-coded decimal */
	uint16_t bcdDevice;

	/** Index of string descriptor describing manufacturer */
	uint8_t  iManufacturer;

	/** Index of string descriptor describing product */
	uint8_t  iProduct;

	/** Index of string descriptor containing device serial number */
	uint8_t  iSerialNumber;

	/** Number of possible configurations */
	uint8_t  bNumConfigurations;
};
```


描述符类型定义的id可以参考lsusb的[libusb_descriptor_type](https://github.com/libusb/libusb/blob/467b6a8896daea3d104958bf0887312c5d14d150/libusb/libusb.h#L283)枚举:

```c
enum libusb_descriptor_type {
	/** Device descriptor. See libusb_device_descriptor. */
	LIBUSB_DT_DEVICE = 0x01,

	/** Configuration descriptor. See libusb_config_descriptor. */
	LIBUSB_DT_CONFIG = 0x02,

	/** String descriptor */
	LIBUSB_DT_STRING = 0x03,

	/** Interface descriptor. See libusb_interface_descriptor. */
	LIBUSB_DT_INTERFACE = 0x04,

	/** Endpoint descriptor. See libusb_endpoint_descriptor. */
	LIBUSB_DT_ENDPOINT = 0x05,

	/** Interface Association Descriptor.
	* See libusb_interface_association_descriptor */
	LIBUSB_DT_INTERFACE_ASSOCIATION = 0x0b,

	/** BOS descriptor */
	LIBUSB_DT_BOS = 0x0f,

	/** Device Capability descriptor */
	LIBUSB_DT_DEVICE_CAPABILITY = 0x10,

	/** HID descriptor */
	LIBUSB_DT_HID = 0x21,

	/** HID report descriptor */
	LIBUSB_DT_REPORT = 0x22,

	/** Physical descriptor */
	LIBUSB_DT_PHYSICAL = 0x23,

	/** Hub descriptor */
	LIBUSB_DT_HUB = 0x29,

	/** SuperSpeed Hub descriptor */
	LIBUSB_DT_SUPERSPEED_HUB = 0x2a,

	/** SuperSpeed Endpoint Companion descriptor */
	LIBUSB_DT_SS_ENDPOINT_COMPANION = 0x30
};
```

可以看到描述符的类型其实不止上面四种,还有很多其他的类型。例如我就能从[UVC 相机终端描述符](https://www.usbzh.com/article/detail-1.html)里面的bmControls字段解析出相机具体支持的操作:

mControls：使用位图来表示支持的视频流。
- D0：扫描模式 //扫描模式(逐行扫描或隔行扫描)
- D1：自动曝光模式
- D2：自动曝光优先级
- D3：曝光时间（绝对值）
- D4：曝光时间（相对）
- D5：焦点（绝对）
- D6：焦点（相对）
- ...

在[libusb](https://github.com/libusb/libusb/blob/467b6a8896daea3d104958bf0887312c5d14d150/libusb/descriptor.c#L165)里面这个UVC相机终端描述符会作为接口描述符的拓展信息保存:

```c
static int parse_interface(libusb_context *ctx,
	struct libusb_interface *usb_interface, const uint8_t *buffer, int size)
{
    ...
    begin = buffer;

    /* Skip over any interface, class or vendor descriptors */
    while (size >= DESC_HEADER_LENGTH) {
        ...
        /* If we find another "proper" descriptor then we're done */
        if (header->bDescriptorType == LIBUSB_DT_INTERFACE ||
            header->bDescriptorType == LIBUSB_DT_ENDPOINT ||
            header->bDescriptorType == LIBUSB_DT_CONFIG ||
            header->bDescriptorType == LIBUSB_DT_DEVICE)
            break;

        buffer += header->bLength;
        parsed += header->bLength;
        size -= header->bLength;
    }

    /* Copy any unknown descriptors into a storage area for */
    /*  drivers to later parse */
    ptrdiff_t len = buffer - begin;
    if (len > 0) {
        void *extra = malloc((size_t)len);
        ...
        memcpy(extra, begin, (size_t)len);
        ifp->extra = extra;
        ifp->extra_length = (int)len;
    }
    ...
}
```

所以在[uvc_scan_control](https://github.com/saki4510t/UVCCamera/blob/master/libuvccamera/src/main/jni/libuvc/src/device.c#L904)里面就从接口描述符的extra信息里面去解析UVC的相关描述符:

```c
uvc_error_t uvc_scan_control(uvc_device_t *dev, uvc_device_info_t *info) {
    ...
    for (interface_idx = 0; interface_idx < info->config->bNumInterfaces; ++interface_idx) {
        if_desc = &info->config->interface[interface_idx].altsetting[0];
        MARK("interface_idx=%d:bInterfaceClass=%02x,bInterfaceSubClass=%02x", interface_idx, if_desc->bInterfaceClass, if_desc->bInterfaceSubClass);
        // select first found Video control
        if (if_desc->bInterfaceClass == LIBUSB_CLASS_VIDEO/*14*/ && if_desc->bInterfaceSubClass == 1) // Video, Control
            break;
        ...
    }
    ...
    buffer = if_desc->extra;
    buffer_left = if_desc->extra_length;

    while (buffer_left >= 3) { // parseX needs to see buf[0,2] = length,type
        block_size = buffer[0];
        parse_ret = uvc_parse_vc(dev, info, buffer, block_size);

        if (parse_ret != UVC_SUCCESS) {
            ret = parse_ret;
            break;
        }

        buffer_left -= block_size;
        buffer += block_size;
    }

    ...
}

uvc_error_t uvc_parse_vc(uvc_device_t *dev, uvc_device_info_t *info,
		const unsigned char *block, size_t block_size) {
	int descriptor_subtype;
	uvc_error_t ret = UVC_SUCCESS;

	UVC_ENTER();

	if (block[1] != LIBUSB_DT_CS_INTERFACE/*36*/) { // not a CS_INTERFACE descriptor??
		UVC_EXIT(UVC_SUCCESS);
		return UVC_SUCCESS; // UVC_ERROR_INVALID_DEVICE;
	}

	descriptor_subtype = block[2];

	switch (descriptor_subtype) {
	case UVC_VC_HEADER:
		ret = uvc_parse_vc_header(dev, info, block, block_size);
		break;
	case UVC_VC_INPUT_TERMINAL:
		ret = uvc_parse_vc_input_terminal(dev, info, block, block_size);
		break;
	case UVC_VC_OUTPUT_TERMINAL:
		break;
	case UVC_VC_SELECTOR_UNIT:
		break;
	case UVC_VC_PROCESSING_UNIT:
		ret = uvc_parse_vc_processing_unit(dev, info, block, block_size);
		break;
	case UVC_VC_EXTENSION_UNIT:
		ret = uvc_parse_vc_extension_unit(dev, info, block, block_size);
		break;
	default:
		LOGW("UVC_ERROR_INVALID_DEVICE:descriptor_subtype=0x%02x", descriptor_subtype);
		ret = UVC_ERROR_INVALID_DEVICE;
	}

	UVC_EXIT(ret);
	return ret;
}
```

只要找到bInterfaceClass等于14,bInterfaceSubClass等于1的视频控制接口,然后在它的拓展信息里面找到`UVC_VC_INPUT_TERMINAL(0x02)`类型的描述符就是我们需要的[UVC 相机终端描述符](https://www.usbzh.com/article/detail-1.html)

# USB通讯

前面有说到`端点是实现USB设备功能的物理缓冲区实体，USB主机和设备是通过端点进行数据交互的`,之前做HID设备通讯的时候流程是找到`bInterfaceClass`为`UsbConstants.USB_CLASS_HID(0x03)`类型的[接口](https://www.usbzh.com/article/detail-64.html),在它下面找到输入端点去写入请求,然后找到输出端点去读取设备响应。

但是UVC的摄像头控制并不是用视频控制接口去读写,而是直接使用USB设备不属于任何接口的0号端口去进行通讯。

例如[uvc_get_pantilt_abs](https://github.com/saki4510t/UVCCamera/blob/c9399e63dfab4b6d260d8cbef92182abc6de0ee0/libuvccamera/src/main/jni/libuvc/src/ctrl.c#L653)里面在传输数据的时候就没有指定端口号:

```c
uvc_error_t uvc_get_pantilt_abs(uvc_device_handle_t *devh, int32_t *pan, int32_t *tilt,
	enum uvc_req_code req_code) {

	uint8_t data[8];
	uvc_error_t ret;

	ret = libusb_control_transfer(devh->usb_devh, REQ_TYPE_GET, req_code,
			UVC_CT_PANTILT_ABSOLUTE_CONTROL << 8,
			devh->info->ctrl_if.input_term_descs->request,
			data, sizeof(data), CTRL_TIMEOUT_MILLIS);

	if (LIKELY(ret == sizeof(data))) {
		*pan = DW_TO_INT(data);
		*tilt = DW_TO_INT(data + 4);
		return UVC_SUCCESS;
	} else {
		return ret;
	}
}
```

因为在[libusb_control_transfer](https://github.com/libusb/libusb/blob/467b6a8896daea3d104958bf0887312c5d14d150/libusb/sync.c#L103)里面调用[libusb_fill_control_transfer](https://github.com/libusb/libusb/blob/467b6a8896daea3d104958bf0887312c5d14d150/libusb/libusb.h#L1918)去填充信息的时候就会把端口指定为0号端口

```c
int API_EXPORTED libusb_control_transfer(libusb_device_handle *dev_handle,
	uint8_t bmRequestType, uint8_t bRequest, uint16_t wValue, uint16_t wIndex,
	unsigned char *data, uint16_t wLength, unsigned int timeout)
{
	...
	libusb_fill_control_transfer(transfer, dev_handle, buffer,
		sync_transfer_cb, &completed, timeout); // 填充transfer信息
	transfer->flags = LIBUSB_TRANSFER_FREE_BUFFER;
	r = libusb_submit_transfer(transfer); // 发送请求
	if (UNLIKELY(r < 0)) {
		libusb_free_transfer(transfer);
		return r;
	}

	sync_transfer_wait_for_completion(transfer); // 等待回复
	...

}

static inline void libusb_fill_control_transfer(
	struct libusb_transfer *transfer, libusb_device_handle *dev_handle,
	unsigned char *buffer, libusb_transfer_cb_fn callback, void *user_data,
	unsigned int timeout)
{
	struct libusb_control_setup *setup = (struct libusb_control_setup *)(void *) buffer;
	transfer->dev_handle = dev_handle;
	transfer->endpoint = 0; // 指定0号端口
	...
}
```

涉及到使用USB进行通讯的4种方式:

- [控制传输](https://www.usbzh.com/article/detail-55.html) - 设备接入主机时，需要通过控制传输去获取USB设备的描述符以及对设备进行识别，在设备的枚举过程中都是使用控制传输进行数据交换。
- [同步传输](https://www.usbzh.com/article/detail-118.html) - 也叫等时传输,用于要求数据连续、实时且数据量大的场合，其对传输延时十分敏感，类似用于USB摄像设备，USB语音设备等等。
- [中断传输](https://www.usbzh.com/article/detail-109.html) - 用于数据量小的数据不连续的但实时性高的场合的一种传输方式，主要应用于人机交互设备(HID)中的USB鼠标和USB键盘等。
- [批量传输](https://www.usbzh.com/article/detail-40.html) - 用于数据量大但对时间要求又不高的场合的一种传输方式，类似用于USB打印机和USB扫描仪等等。

## 控制传输

控制传输是usb设备一定会支持的传输方式,因为描述符就是通过这种方式获取的.

在安卓应用层我们可以通过调用UsbDeviceConnection.controlTransfer来实现,参考UVCCamera里面[uvc_get_pantilt_abs](https://github.com/saki4510t/UVCCamera/blob/c9399e63dfab4b6d260d8cbef92182abc6de0ee0/libuvccamera/src/main/jni/libuvc/src/ctrl.c#L653)里面获取[PanTilt值](https://www.usbzh.com/article/detail-47.html#13.PanTilt%EF%BC%88%E7%BB%9D%E5%AF%B9%EF%BC%89%E6%8E%A7%E5%88%B6)的c代码,在java层可以用下面代码获取

```kotlin
private const val CONTROL_REQ_TYPE_GET = 0xa1
private const val UVC_GET_CUR = 0x81

val connection = usbManager.openDevice(device)

// 先claim bInterfaceClass为CC_VIDEO(0x0E) bInterfaceSubClass为SC_VIDEOCONTROL(0x01)的摄像头控制接口
val vcInterface = UsbUtils.getInterface(device, UsbConstants.USB_CLASS_VIDEO, USB_SUBCLASS_VIDEO_CONTROL)
connection.claimInterface(vcInterface, true)

// 然后发送控制指令获取PanTilt绝对值
val buff = ByteArray(8)
val index = getPanTiltControlIndex(connection)
val value = CT_PANTILT_ABSOLUTE_CONTROL.shl(8)
connection.controlTransfer(CONTROL_REQ_TYPE_GET, UVC_GET_CUR, value, index, buff, buff.size, 100)

// buff前四个byte组合起来是pan值
// buff后四个byte组合起来是tilt值
val pan = bytes[0].toUByte().toInt().shl(0) or
bytes[1].toUByte().toInt().shl(8) or
bytes[2].toUByte().toInt().shl(16) or
bytes[3].toUByte().toInt().shl(24)

val tilt = bytes[4].toUByte().toInt().shl(0) or
bytes[5].toUByte().toInt().shl(8) or
bytes[6].toUByte().toInt().shl(16) or
bytes[7].toUByte().toInt().shl(24)

connection.releaseInterface(usbInterface)
connection.close()
```

这里解释下上面的值如何来的,首先看[GET_CUR](https://www.usbzh.com/article/detail-29.html#%E8%8E%B7%E5%8F%96%E8%AF%B7%E6%B1%82GET_CUR)的文档:

|requestType|request|value|index|buffer|length|
|-|-|-|-|-|-|
|10100001(接口或实体)<br/>— — — — —<br/>10100010（端点）|GET_CUR<br/>GET_MIN<br/>GET_MAX<br/>GET_RES<br/>GET_LEN<br/>GET_INFO<br/>GET_DEF|UVC中大多数情况下取值都为控制选择器CS(高字节)，低字节为零。当实体ID取不同值时则该字段取值也会有所不同|实体ID(高字节)、接口(低字节)<br/>— — — — — <br/>端点(低字节)|用来接收数据或者发送数据的buffer|buffer的大小|

### value

例如我们现在要获取PanTilt的绝对值,那么在[value字段部分文档](https://www.usbzh.com/article/detail-29.html#wValue%E5%AD%97%E6%AE%B5(2%E5%AD%97%E8%8A%82))里面可以看到当Entity ID值为Camera Terminal时：

|ControlSelector|Value|
|-|-|
|...|...|
|CT_PANTILT_ABSOLUTE_CONTROL|0x0D|
|...|...|

又因为`value的值为控制选择器CS(高字节)，低字节为零。`所以value的值应该是`0x0D << 8`

### index

而`Entity ID值为Camera Terminal`指的是[终端描述符](https://www.usbzh.com/article/detail-1.html)的bTerminalID, 由于它属于[控制接口描述符](https://www.usbzh.com/article/detail-59.html)的extra信息,所以还需要指的该接口的bInterfaceNunber:

```kotlin
val USB_DESC_TYPE_INTERFACE_LEN = 9.toByte()
val USB_DESC_TYPE_INTERFACE = 0x04.toByte()
val USB_DESC_TYPE_CS_INTERFACE = 0x24.toByte()
val USB_DESC_SUB_TYPE_VC_INPUT_TERMINAL = 0x02.toByte()
val USB_SUBCLASS_VIDEO_CONTROL = 0x01

private fun getPanTiltControlIndex(connection: UsbDeviceConnection): Int {
    val desc = connection.rawDescriptors ?: return -1

    var index = 0
    var isInVideoControlInterface = false
    var interfaceNumber = 0
    while (index < desc.size) {
        if (desc[index] == USB_DESC_TYPE_INTERFACE_LEN
            && desc[index + 1] == USB_DESC_TYPE_INTERFACE
            && desc[index + 5] == UsbConstants.USB_CLASS_VIDEO.toByte()
            && desc[index + 6] == USB_SUBCLASS_VIDEO_CONTROL.toByte()
        ) {
            // 找到bInterfaceClass为CC_VIDEO(0x0E) bInterfaceSubClass为SC_VIDEOCONTROL(0x01)的摄像头控制接口的interfaceNumber
            isInVideoControlInterface = true
            interfaceNumber = desc[index + 2].toInt()
        } else if (isInVideoControlInterface) {
            if (desc[index + 1] != USB_DESC_TYPE_CS_INTERFACE) {
                return -1
            }
            if (desc[index + 2] == USB_DESC_SUB_TYPE_VC_INPUT_TERMINAL) {
                // 在摄像头控制接口下找到bDescriptorType为CS_INTERFACE(0x24) bDescriptorSubtype为VC_INPUT_TERMINAL(0x02)的摄像头终端描述符
                // 获取它的bTerminalID用来和前面获取到的摄像头控制接口的interfaceNumber拼接成index
                return desc[index + 3].toInt().shl(8).or(interfaceNumber)
            }
        }

        index += desc[index]
    }
    return -1
}
```

### request

我们要获取的是当前值所以request是[GET_CUR](https://www.usbzh.com/article/detail-39.html)(0x81),其他值的定如下:

|名称|值|说明|
|-|-|-|
|RC_UNDEFINED|0x00|未定义|
|SET_CUR|0x01|设置属性|
|GET_CUR|0x81|获取当前属性|
|GET_MIN|0x82|获取最小设置属性|
|GET_MAX|0x83|获取最大设置属性|
|GET_RES|0x84|获取分辨率属性|
|GET_LEN|0x85|获取数据长度属性|
|GET_INF|0x86|获取设备支持的特定类请求属性|
|GET_DEF|0x87|获取默认属性|

### requestType

最后再来看requestType,由于index需要选择的是`实体ID(高字节)、接口(低字节)`所以requestType应该是`10100001(接口或实体)`。它的值这么奇怪是因为requestType的每个bit都是有意义的:

{% img /AndroidUvcControl/1.png %}

由于命令接受者为接口,所以我们在发送控制指令前还是需要找到这个接口用claimInterface去锁定它。

### 使用uvc控制指令的坑

似乎是因为使用安卓的Camera2等接口去读取摄像头画面的时候会使用到这个[控制接口](https://www.usbzh.com/article/detail-59.html),所以如果在预览的时候去claimInterface锁定它就会造成画面卡死。

看起来似乎需要完全使用uvc自己从[视频流接口](https://www.usbzh.com/article/detail-68.html)读取画面,而不能一半用uvc去控制摄像头另一半用安卓原生api去获取预览画面。或者用取巧的方法在发送控制指令的时候先停止预览,发送完再开始。

## 其他三种传输

其他三种传输都是需要找到对应的端点才能进行通讯的,所以需要先获取到端点信息.用UsbInterface.getEndpoint去遍历接口下的端点,然后判断端点的类型和读写方向:

```kotlin
for (i in 0 until usbInterface.endpointCount) {
    val usbEndpoint = usbInterface.getEndpoint(i)
    when (usbEndpoint.type) {
        UsbConstants.USB_ENDPOINT_XFER_BULK -> {
            // 批量传输
            if (usbEndpoint.direction == UsbConstants.USB_DIR_OUT) {
                // 可写入端点
            } else if (usbEndpoint.direction == UsbConstants.USB_DIR_IN) {
                // 可读取端点
            }
        }
        UsbConstants.USB_ENDPOINT_XFER_ISOC -> {
            // 中断传输
        }
        UsbConstants.USB_ENDPOINT_XFER_INT -> {
            // 同步传输
        }
    }
}
```

他们最终都是通过UsbDeviceConnection.bulkTransfer去调用的,例如可以先写入请求在读取响应:

```kotlin
val requestBuffer = ByteArray(256)
// 将数据保存到requestBuffer
// 然后往写入端点写入请求数据
connection.bulkTransfer(outPoint, sendBuff, sendBuff.size, timeout)

// 从读取端点读取响应
val responseBuffer = ByteArray(256)
connection.bulkTransfer(inPoint, responseBuffer, responseBuffer.size, timeout)
```
