---
layout: post
title: Android BLE4.0(设备搜索)
date: 2016-12-25 20:45:00 +0800
description:  # Add post description (optional)
tags: BLE4.0
---
接上一篇[Android BLE4.0(基本知识)](http://www.jianshu.com/p/dc67e6fe036b)，
本篇记录在Android中的蓝牙4.0开发。要想与蓝牙设备进行通讯，
首先要连接到相应的设备，连接到相应的设备之前，
我们要能够搜索到它。所以我们先从找到设备开始。

## 1、申请权限

在Android中要想使用蓝牙，需要添加以下两个权限

```
<uses-permission android:name="android.permission.BLUETOOTH"/>
<uses-permissionandroid:name="android.permission.BLUETOOTH_ADMIN"/>
```

一般情况下，添加上面两个权限应该是可以了，
但是老司机们都应该知道Android 6.0采用新的权限机制来保护用户的隐私，
将权限分为**Normal Permissions** *(不涉及用户隐私，不需要用户授权)*和
**Dangerous Permission** *(涉及用户隐私，使用时需要用户实时授权)*两种。

蓝牙权限本身不属于用户隐私的权限，
但是**在Android 6.0之后要用蓝牙还需要添加一个模糊定位的权限**

```
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
```

这个权限就属于隐私权限的范围了。。。。一股淡淡的忧伤。。。所以想要兼容6.0还要在代码中检测权限：


### 1.1、Android 6.0 检测并申请权限

```
/*
 * 检测并申请权限
 */
private void checkBluetoothPermission() {
    if (Build.VERSION.SDK_INT >= 23) {
        //校验是否已具有模糊定位权限
        if (ContextCompat.checkSelfPermission(TYMposActivity.this,
                Manifest.permission.ACCESS_COARSE_LOCATION)
                != PackageManager.PERMISSION_GRANTED) {
           //申请模糊定位权限
           ActivityCompat.requestPermissions(TYMposActivity.this,
                   new String[]{Manifest.permission.ACCESS_COARSE_LOCATION},
                MY_PERMISSIONS_REQUEST_ACCESS_COARSE_LOCATION);
        } else {
           //具有权限
            connectBluetooth();
        }
    } else {
         //系统不高于6.0直接执行
         connectBluetooth();
    }
}
```

这里用到了两个API
- **ContextCompat.checkSelfPermission**
主要用于检测某个权限是否已经被授予，  
返回值为**PackageManager.PERMISSION_DENIED**
或者是**PackageManager.PERMISSION_GRANTED**。  
当返回DENIED就需要进行申请授权了

- **ActivityCompat.requestPermissions**
这是个异步方法，有三个参数  
第一个参数是Context，这个不多说；  
第二个参数是需要申请的权限的字符串数组；  
第三个参数为requestCode，主要用于回调的时候检测。  
可以从方法名requestPermissions以及第二个参数看出，是支持一次性申请多个权限的，  
系统会通过对话框 **逐一** 询问用户是否授权。

### 1.2、授权返回处理：

对授权返回值进行处理，有点类似于startActivityForResult

```
@Override
public void onRequestPermissionsResult(int requestCode,
        String permissions[], int[] grantResults) {
    switch (requestCode) {
        case MY_PERMISSIONS_REQUEST_ACCESS_COARSE_LOCATION: {
            // 如果请求被取消，则结果数组为空.
            if (grantResults.length > 0
                && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
                //已授权
            } else {
                // 未授权
            }
            return;
        }
    }
}
```

关于6.0的权限申请到此结束，另外补充一下：
如果想声明你的app只为具有BLE的设备提供，在manifest文件中包括：

```
<uses-feature android:name="android.hardware.bluetooth_le" 
android:required="true"/>
```

但如果想让你的app提供给那些不支持BLE的设备，
需要在manifest中包括上面代码并设置required="false"，
然后在运行时可以通过使用PackageManager.hasSystemFeature()确定BLE的可用性。

## 2、检查设备是否支持蓝牙

*注意：Android系统版本在4.3及以上才能使用蓝牙4.0。*  
使用此检查确定BLE是否支持在设备上，然后你可以有选择性禁用BLE相关的功能

```
if (!getPackageManager().hasSystemFeature(PackageManager.FEATURE_BLUETOOTH_LE)) {
    Toast.makeText(this, "设备不支持蓝牙4.0", Toast.LENGTH_SHORT).show();
    finish();
}
```

## 3、蓝牙搜索

大致步骤

- **获取BluetoothManager**
- **获取BluetoothAdapter**
- **调用BluetoothAdapter.startLeScan开始扫描设备**

**获取BluetoothManager：**

```
BluetoothManager manager = (BluetoothManager)   
context.getSystemService(Context.BLUETOOTH_SERVICE);
```

**获取BluetoothAdapter**
获取BluetoothAdapter有两种方式：
- 第一种：
```
BluetoothAdapter adapter = manager.getAdapter();
```
- 第二种
```
BluetoothAdapter adapter = BluetoothAdapter.getDefaultAdapter(); 
```

**扫描附近BLE设备**

在开始扫描之前先打开蓝牙,打开之前需要先判断蓝牙是否已经打开：

```
/**
 * 蓝牙是否打开
 *
 * @return
 */
public boolean isBleOpen() {
   if (adapter == null) {
      return false;
   }
   return adapter.isEnabled();
}
```

**打开蓝牙有两种方式**

-  跳转到系统的界面，手动打开

    ```
    /**
    * 系统打开蓝牙
    */
    public void sysOpenBLE(Activity activity, int requestCode) {
        Intent intent = new Intent(BluetoothAdapter.ACTION_REQUEST_ENABLE);
        activity.startActivityForResult(intent, requestCode);
    }
    ```
- 自动打开蓝牙

    ```
    /**
    * 打开蓝牙
    */
    public boolean openBLE() {
        if (adapter == null) {
        return false;
    }
    return adapter.enable();
    }
    ```

确保蓝牙打开之后开始扫描设备：
通过adapter.startLeScan()方法来进行扫描。这里有两个方法：

```
/**
 *
 */
public boolean startLeScan(LeScanCallback callback) {
    return startLeScan(null, callback);
}

/**
 * @param serviceUuids Array of services to look for
 * 需要过滤的UUID服务，扫描的时候，只返回与指定UUID相同的设备
 */
public boolean startLeScan(final UUID[] serviceUuids, final LeScanCallback callback) {
    if (DBG) Log.d(TAG, "startLeScan(): " + Arrays.toString(serviceUuids));
    if (callback == null) {
        if (DBG) Log.e(TAG, "startLeScan: null callback");
        return false;
    }
    BluetoothLeScanner scanner = getBluetoothLeScanner();
    if (scanner == null) {
        if (DBG) Log.e(TAG, "startLeScan: cannot get BluetoothLeScanner");
        return false;
    }
  ......
}
```

以上代码摘自Android源码，这两个不多说什么，
重点在 **serviceUuids**这里，大家都知道每个蓝牙都有一个服务UUID,
这个参数就是针对这个UUID进行过滤的，扫描返回结果的时候，
只会返回与UUID相同的设备，另外一个*是*LeScanCallback**，
这个是返回搜索结果的回调，如果这个LeScanCallback不传，扫描是不会启动的。

```
BluetoothAdapter.LeScanCallback scanCallback = new BluetoothAdapter.LeScanCallback() {
   @Override
   public void onLeScan(BluetoothDevice bluetoothDevice, int rssi, byte[] scanRecord) {
     
   }
};
```

在这个扫描回调中，回调了三个参数：

- **BluetoothDevice**   

```

/**
 * Represents a remote Bluetooth device. A {@link BluetoothDevice} lets you
 * create a connection with the respective device or query information about
 * it, such as the name, address, class, and bonding state.
 * 蓝牙信息相关的类，可以获取蓝牙的名称，地址，已经绑定状态等
 */
public final class BluetoothDevice implements Parcelable {

.....
//绑定状态相关
/**
 * Indicates the remote device is not bonded (paired).
 * <p>There is no shared link key with the remote device, so communication
 * (if it is allowed at all) will be unauthenticated and unencrypted.
 */
public static final int BOND_NONE = 10;
/**
 * Indicates bonding (pairing) is in progress with the remote device.
 */
public static final int BOND_BONDING = 11;
/**
 * Indicates the remote device is bonded (paired).
 * <p>A shared link keys exists locally for the remote device, so
 * communication can be authenticated and encrypted.
 * <p><i>Being bonded (paired) with a remote device does not necessarily
 * mean the device is currently connected. It just means that the pending
 * procedure was completed at some earlier time, and the link key is still
 * stored locally, ready to use on the next connection.
 * </i> 
*/
public static final int BOND_BONDED = 12;

.....

//设备类型
/**
 * Bluetooth device type, Unknown
 */
public static final int DEVICE_TYPE_UNKNOWN = 0;
/**
 * Bluetooth device type, Classic - BR/EDR devices，传统
 */
public static final int DEVICE_TYPE_CLASSIC = 1;
/**
 * Bluetooth device type, Low Energy - LE-only ，低功耗
 */
public static final int DEVICE_TYPE_LE = 2;
/**
 * Bluetooth device type, Dual Mode - BR/EDR/LE 双模式
 */
public static final int DEVICE_TYPE_DUAL = 3;

............

}
```

其实就是蓝牙的设备的一个实体类。没什么可以说的。

- **rssi**
RSSI的值作为对远程蓝牙设备信号值，正常为负值;  值越大信号越强;
- **scanRecord**
远程设备提供的配对号，一般用不到。


## 小结
如果不出意外，到这里是可以扫描到周边的蓝牙设备了，
对于蓝牙开发已经算是迈出了第一步，
也有了一个初步了解，接下来记录怎么连接到一个搜索到的设备。