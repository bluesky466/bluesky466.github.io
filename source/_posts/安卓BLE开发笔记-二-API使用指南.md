title: 安卓BLE开发笔记(二) API使用指南
date: 2021-10-08 19:06:40
tags:
  - 技术相关
  - Android
---

[上一篇](https://blog.islinjw.cn/2021/09/29/%E5%AE%89%E5%8D%93BLE%E5%BC%80%E5%8F%91%E7%AC%94%E8%AE%B0-%E4%B8%80-BLE%E5%8D%8F%E8%AE%AE%E5%85%A5%E9%97%A8/)简单介绍了Ble协议，这篇来看看安卓上的代码具体要怎么写。

# 权限与功能

要在代码中使用蓝牙功能需要先申请到对应的权限，在AndroidManifest.xml文件中添加权限声明

```xml
<uses-permission android:name="android.permission.BLUETOOTH" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
```

Ble属于蓝牙的功能，所以我们需要打开手机的蓝牙功能

```kotlin
// 打开蓝牙功能
val bluetoothManager = getSystemService(Context.BLUETOOTH_SERVICE) as BluetoothManager
if (!bluetoothManager.adapter.isEnabled) {
    registerForActivityResult(ActivityResultContracts.StartActivityForResult()) {
        startScan()
    }.launch(Intent(BluetoothAdapter.ACTION_REQUEST_ENABLE))
    return
}
```

而且虽然我们使用的是蓝牙功能，但是除了蓝牙的相关权限之外，还需要额外申请一个模糊定位的权限。要不然会启动蓝牙扫描失败:

> 09-11 09:33:47.143 13249 13292 I BtGatt.ScanManager: Cannot start unfiltered scan in location-off. This scan will be resumed when location is on: 6

原因是我们可以通过检查周边蓝牙设备信号的强度，来实现模糊定位。这个权限是需要动态申请的:

```kotlin
// 判断是否有模糊定位的权限
if (PERMISSION_DENIED == PermissionChecker.checkCallingOrSelfPermission(this, ACCESS_COARSE_LOCATION)) {
    requestPermissions(arrayOf(ACCESS_COARSE_LOCATION), REQUEST_CODE_ACCESS_COARSE_LOCATION)
    return
}

// 判断GPS是否打开
val locationManager = getSystemService(LOCATION_SERVICE) as LocationManager
if (!locationManager.isProviderEnabled(LocationManager.GPS_PROVIDER)) {
    Toast.makeText(this, "GPS功能没有打开", Toast.LENGTH_SHORT).show()
    return
}
```

PS : 我司的大板上实际上没有GPS功能，所以最终让系统哥在framework里面把这个判断去掉了:

```java
// packages/apps/Bluetooth/src/com/android/bluetooth/gatt/ScanManager.java
// 将这个if判断直接删掉就好，这里只是用来做安全判断的，实际上蓝牙的功能并不依赖gps
if (!locationEnabled && !isFiltered) {
    Log.i(TAG, "Cannot start unfiltered scan in location-off. This scan will be"
            + " resumed when location is on: " + client.scannerId);
    mSuspendedScanClients.add(client);
    if (client.stats != null) {
        client.stats.recordScanSuspend(client.scannerId);
    }
    return;
}
```

# 搜索蓝牙设备

功能和权限正常之后我们就能开始扫描周边的蓝牙设备。我看网上很多的教程都是通过BluetoothAdapter.startLeScan去实现的，实际上这个方法已经被标记为过时了，我们这里用新的api去实现:

```Kotlin
object : ScanCallback() {
    override fun onScanResult(callbackType: Int, result: ScanResult?) {
        super.onScanResult(callbackType, result)
        // result里面可以拿到信号强度rssi、设备信息device、广播信息scanRecord等
    }
}

val manager = context.getSystemService(Context.BLUETOOTH_SERVICE) as BluetoothManager
val adapter = manager.adapter
scanner = adapter?.bluetoothLeScanner ?: return false
scanner.startScan(callback)
```

# 连接蓝牙设备

蓝牙设备的连接直接调用onScanResult的result.device的connect方法即可:

```kotlin
private var gatt: BluetoothGatt? = null
private val gattCallback = object : BluetoothGattCallback() {
    override fun onConnectionStateChange(gatt: BluetoothGatt?, status: Int, newState: Int) {
        super.onConnectionStateChange(gatt, status, newState)
        this@BleHelper.gatt = gatt ?: return

        if (newState == BluetoothProfile.STATE_CONNECTED) {
            connectCallback?.onConnected(gatt)
        }
    }
    ...
}

fun connectDevice(context: Context,device: BluetoothDevice,callback: ConnectCallback) {
    connectCallback = callback
    device.connectGatt(context, true, gattCallback)
}
```

连接成功之后我们需要手动去扫描Ble设备提供的服务:

```kotlin
private var gatt: BluetoothGatt? = null
private val gattCallback = object : BluetoothGattCallback() {
    ...
    override fun onServicesDiscovered(gatt: BluetoothGatt?, status: Int) {
        super.onServicesDiscovered(gatt, status)
        val services = gatt?.services ?: return
        discoverServicesCallback?.onServicesDiscovered(services)
    }
    ...
}

fun discoverServices(gatt: BluetoothGatt, callback: DiscoverServicesCallback) {
    discoverServicesCallback = callback
    gatt.discoverServices()
}
```

可以看到，连接和服务发现都是通过BluetoothGattCallback来回调的。实际上所有的GATT相关操作者如读写Characteristic、修改MTU等都是在这里回调的。

# Characteristic读写与监听

设备服务发现成功之后，我们就可以在BluetoothGatt里面遍历services和其中的characteristics。当然我们也可以通过uuid去搜索指定的characteristic:

```kotlin
private var gatt: BluetoothGatt? = null
fun getCharacteristic(serviceUuid: String, characteristicUuid: String): BluetoothGattCharacteristic? {
    return gatt
        ?.getService(UUID.fromString(serviceUuid))
        ?.getCharacteristic(UUID.fromString(characteristicUuid))
}
```

通过characteristic的properties属性我们可以判断到characteristic是不是可读、可写或者可监听的:

```kotlin
fun isCharacteristicReadable(characteristic: BluetoothGattCharacteristic): Boolean {
    return (characteristic.properties and BluetoothGattCharacteristic.PROPERTY_READ) != 0
}

fun isCharacteristicWriteable(characteristic: BluetoothGattCharacteristic): Boolean {
    return (characteristic.properties and BluetoothGattCharacteristic.PROPERTY_WRITE) != 0
}

fun isCharacteristicNotify(characteristic: BluetoothGattCharacteristic): Boolean {
    return (characteristic.properties and BluetoothGattCharacteristic.PROPERTY_NOTIFY) != 0
}
```

## 读取characteristic

读取characteristic很简单，直接调用BluetoothGatt.readCharacteristic方法即可:

```kotlin
private var gatt: BluetoothGatt? = null
private val gattCallback = object : BluetoothGattCallback() {
    ...
    override fun onCharacteristicRead(
        gatt: BluetoothGatt?,
        characteristic: BluetoothGattCharacteristic?,
        status: Int
    ) {
        super.onCharacteristicRead(gatt, characteristic, status)
        //characteristic?.value 即为读取出来的具体ByteArray
    }
    ...
}

fun readCharacteristic(characteristic: BluetoothGattCharacteristic, callback: ReadCharacteristicCallback) {
    readCharacteristicCallback = callback
    gatt?.readCharacteristic(characteristic)
}
```

## 写入characteristic

写入的话上一篇有提到过安卓上默认MTU为23，最多一次只能写入20个字节的数据，所以如果数据量比较大的话需要设置一下mtu:

```kotlin
private var gatt: BluetoothGatt? = null
private val gattCallback = object : BluetoothGattCallback() {
        ...
        override fun onMtuChanged(gatt: BluetoothGatt?, mtu: Int, status: Int) {
            super.onMtuChanged(gatt, mtu, status)
            Log.d(TAG, "onMtuChanged $mtu")
        }
        ...
}

fun setMtu(mtu: Int) {
    gatt?.requestMtu(mtu)
}
```

等MTU设置成功之后再调用BluetoothGatt.writeCharacteristic方法写入

```kotlin
private var gatt: BluetoothGatt? = null
private val gattCallback = object : BluetoothGattCallback() {
        ...
        override fun onCharacteristicWrite(
            gatt: BluetoothGatt?,
            characteristic: BluetoothGattCharacteristic?,
            status: Int
        ) {
            super.onCharacteristicWrite(gatt, characteristic, status)
            Log.d(TAG, "onCharacteristicWrite ${characteristic?.uuid} ${status}")
        }
        ...
}

fun writeCharacteristic(characteristic: BluetoothGattCharacteristic, bytes: ByteArray) {
    characteristic.value = bytes
    gatt?.writeCharacteristic(characteristic)
}
```

## 监听characteristic

监听characteristic，如上一篇所说，需要先修改其Client Characteristic Configuration Descriptor(uuid 0x2902):

```kotlin
private var gatt: BluetoothGatt? = null

fun startCharacteristicChangeNotify(characteristic: BluetoothGattCharacteristic, callback: CharacteristicChangedCallback) {
    val gatt = gatt ?: return
    characteristicChangedCallback = callback
    if (gatt.setCharacteristicNotification(characteristic, true)) {
        val descriptor = characteristic.getDescriptor(
            UUID.fromString(UUID_DESCRIPTOR_CLIENT_CHARACTERISTIC_CONFIGURATION)
        )
        descriptor.value = BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE
        gatt.writeDescriptor(descriptor)
    }
}
```

然后才能接受到值改变的回调:

```kotlin
private val gattCallback = object : BluetoothGattCallback() {
    ...
    override fun onCharacteristicChanged(
        gatt: BluetoothGatt?,
        characteristic: BluetoothGattCharacteristic?
    ) {
        super.onCharacteristicChanged(gatt, characteristic)
        if (characteristic != null) {
            characteristicChangedCallback?.onCharacteristicChanged(characteristic)
        }
    }
    ...
}
```

# 配对

经典蓝牙的配对是通过BluetoothDevice.createBond()方法实现的，如果在扫描到设备之后直接调用，则可以配对成功。但如果先使用ble connect连接上去之后再去调用就会失败。据说是因为BluetoothDevice.createBond内部也会先判断是否connect，如果connect成功就不继续(这里我没有去找实际的源码验证，存疑)。

但是我发现在原生的Setting蓝牙设置里面，就算是我已经ble connect成功的设备，再去点击连接也是可以绑定的（这里用了我司的一个智能笔设备验证）。于是直接查看原生Setting源码。发现可以通过bluetooth服务去实现:

```kotlin
private const val PACKAGE_BLUETOOTH = "com.android.bluetooth"
private const val ACTION_BLUETOOTH = "android.bluetooth.IBluetoothHidHost"


private var hidHost: IBluetoothHidHost? = null


fun init(application: Application) {
    this.application = application

    val intent = Intent(ACTION_BLUETOOTH)
    intent.setPackage(PACKAGE_BLUETOOTH)

    Log.d(TAG, "init")
    application.bindService(
        intent,
        object : ServiceConnection {
            override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
                Log.d(TAG, "IBluetoothHidHost onServiceConnected")
                hidHost = IBluetoothHidHost.Stub.asInterface(service)
            }

            override fun onServiceDisconnected(name: ComponentName?) {
                Log.d(TAG, "IBluetoothHidHost onServiceDisconnected")
            }
        },
        Context.BIND_AUTO_CREATE
    )
}

/**
 * 绑定设备
 */
fun bond(device: BluetoothDevice) {
    val result = hidHost?.connect(device)
    Log.d(TAG,"bond $result")
}
```

IBluetoothHidHost实际上在AndroidSdk内部没有提供，可以直接拷贝系统源码中生成的aidl代码来使用。

# 完整Demo

完整demo已经上传到[github](https://github.com/bluesky466/BleDemo)感兴趣的同学可以clone下来参考