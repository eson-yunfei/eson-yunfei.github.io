---
layout: post
title: Android BLE4.0(设备连接)
date: 2017-02-21 15:37:00 +0800
description:  # Add post description (optional)
tags: BLE4.0
---
这段时间比较忙，就没有顾上更新文章，废话就到这里，
言归正传，
接上一篇 [Android BLE4.0(设备搜索)](http://www.jianshu.com/p/692b735d64fb),
说一下Android如何连接蓝牙外围设备。

在上一篇文章中说道，扫描设备后会得到一 `BluetoothDevice`的一个实例，
我们都知道每一个BluetoothDevice都代表一个蓝牙设备。

那么，拿到这个设备之后，有三个问题：

- 如何保证连接的设备的唯一性。
- 如何与设备进行连接。
- 连接成功之后，如何进行通信。

那就围绕这三个问题，一步一步来吧。

## 一、保证设备的唯一性

保证设备唯一性，很多人开发者都会遇到相关的问题。
关于怎么解决这个问题，每一个开发者或者公司都有自己的解决方案，
这个就不具体讨论了。大部分的开发者认为设备的**MAC地址**是一个不错的选择，
确实也是这样。蓝牙设备同样也是有MAC地址的，通过MAC地址去连接设备，
应该能保证设备的唯一性吧。

## 二、如何与设备进行连接

解决了第一个问题，我们知道了应该用设备的MAC地址去连接，
具体要怎么做呢？说这个问题之前，先解决另外一个问题：

### 2.1、*大家都说习惯说连接设备。连接设备，到底连接的是什么？*

这就要说一下之前在
[Android BLE4.0(基本知识)](http://www.jianshu.com/p/dc67e6fe036b)
一篇中介绍的几个概念（没有看过的可以去看一下）。
其中有一个Generic Attribute Profile (GATT)的东西。
每一个蓝牙设备都有一个GATT 的服务端，
我们所说的连接设备，其实是连接到设备的GATT 的服务端。

```
mBluetoothGatt = device.connectGatt(this, false, mGattCallback);
```

### 2.2、如何连接？

**“代码是最好的老师”**。我们就拿[Google的官方蓝牙Demo](https://github.com/googlesamples/android-BluetoothLeGatt)    
进行示例解说（其实也是自己学习一下）。

拿到源码之后，先大致浏览一下文件目录结构：

![文件目录.png](http://upload-images.jianshu.io/upload_images/2825667-843c4d168917001e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

上图就是例子中的文件目录，做过Android蓝牙开发的应该对他不陌生吧？
文件不多，但每一个作用都很重要。因为本篇文章主要是说蓝牙连接的部分，
我这里就只拿*DeviceControlActivity* 和 *BluetoothLeService* 这两个文件来说。
没有做过蓝牙的同学有可能会问，为什么只拿这两个文件来说，
我只想想说：看文件名，不解释。

我们一步一步来，先从DeviceControlActivity入手：

```
private String mDeviceName;
private String mDeviceAddress;
private BluetoothLeService mBluetoothLeService;
private ArrayList<ArrayList<BluetoothGattCharacteristic>> mGattCharacteristics =
        new ArrayList<ArrayList<BluetoothGattCharacteristic>>();
private BluetoothGattCharacteristic mNotifyCharacteristic;

@Override
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    setContentView(R.layout.gatt_services_characteristics);

    final Intent intent = getIntent();
    mDeviceName = intent.getStringExtra(EXTRAS_DEVICE_NAME);
    mDeviceAddress = intent.getStringExtra(EXTRAS_DEVICE_ADDRESS);

    ...

    Intent gattServiceIntent = new Intent(this, BluetoothLeService.class);
    bindService(gattServiceIntent, mServiceConnection, BIND_AUTO_CREATE);
}
```

首先，
在`onCreate()`的里面接收了`mDeviceName` 和 `mDeviceAddress` 两个值，
至于这两个值哪里来的，我就不用说了吧。
然后调用`bindService()`启动了`BluetoothLeService`。
还有一个Service连接状态的回调（可以这么理解吧）： `mServiceConnection`。

当onServiceConnected()触发的时候，
调用了BluetoothLeService里面的connect()方法。

```
// Code to manage Service lifecycle.
private final ServiceConnection mServiceConnection = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName componentName, IBinder service) {

        mBluetoothLeService = ((BluetoothLeService.LocalBinder) service).getService();
        if (!mBluetoothLeService.initialize()) {
            Log.e(TAG, "Unable to initialize Bluetooth");
            finish();
        }

        // Automatically connects to the device upon successful start-up initialization.
        //成功启动初始化后自动连接到设备。
        mBluetoothLeService.connect(mDeviceAddress);
    }
    @Override
    public void onServiceDisconnected(ComponentName componentName) {
        mBluetoothLeService = null;
    }
};
```

看到这里，我们大概知道，最核心的应该是在connect()里面了，
按照代码的执行顺序，进入到BluetoothLeService里面，
找到connect()方法，我们看一下源代码：

```
public boolean connect(final String address) {
    if (mBluetoothAdapter == null || address == null) {
        Log.w(TAG, "BluetoothAdapter not initialized or unspecified address.");
        return false;
    }
    // Previously connected device.  Try to reconnect.
    if (mBluetoothDeviceAddress != null && address.equals(mBluetoothDeviceAddress)
            && mBluetoothGatt != null) {
        Log.d(TAG, "Trying to use an existing mBluetoothGatt for connection.");
        if (mBluetoothGatt.connect()) {
            mConnectionState = STATE_CONNECTING;
            return true;
        } else {
            return false;
        }
    }
    final BluetoothDevice device = mBluetoothAdapter.getRemoteDevice(address);
    if (device == null) {
        Log.w(TAG, "Device not found.  Unable to connect.");
        return false;
    }
    // We want to directly connect to the device, so we are setting the autoConnect
    // parameter to false.
    mBluetoothGatt = device.connectGatt(this, false, mGattCallback);
    Log.d(TAG, "Trying to create a new connection.");
    mBluetoothDeviceAddress = address;
    mConnectionState = STATE_CONNECTING;
    return true;
}
```

可以看到，这个方法需要传入了一个`address`，
这个address就是指的设备的MAC地址，这就印证了第一个问题。
通过mBluetoothAdapter获取到一个BluetoothDevice，
然后在调用device.connectGatt()，即可连接到此设备。

这里有三个参数
+ Context  
+ autoConnect ：是否自动连接
+ BluetoothGattCallback ：用于向客户端传递结果，如连接状态等等。

```
private final BluetoothGattCallback mGattCallback = new BluetoothGattCallback() {
    @Override
    public void onConnectionStateChange(BluetoothGatt gatt, int status, int newState) {
        String intentAction;
        if (newState == BluetoothProfile.STATE_CONNECTED) {
            
            Log.i(TAG, "Connected to GATT server.");
           
        } else if (newState == BluetoothProfile.STATE_DISCONNECTED) {
           
            Log.i(TAG, "Disconnected from GATT server.");
        }
    }
    @Override
    public void onServicesDiscovered(BluetoothGatt gatt, int status) {
       
    }
    @Override
    public void onCharacteristicRead(BluetoothGatt gatt,   
    BluetoothGattCharacteristic characteristic,  
    int status) {
        
    }
    @Override
    public void onCharacteristicChanged(BluetoothGatt gatt,  
    BluetoothGattCharacteristic characteristic) {
       
    }
};
```

### *PS：*
这篇主要记录连接蓝牙部分，文章在2017年1月份就已经开始着手写了，
本来是打算春节前发布的，由于各种原因吧，一直拖到2月底。。。。

当时想着把与蓝牙通讯部分也写进去的，现在我又重新整理了一下，
觉得还是要把蓝牙通讯单独列出一篇。就说这么多吧。