---
layout: post
title: Android BLE4.0(蓝牙通信)
date: 2017-02-24 09:33:00 +0800
description:  # Add post description (optional)
tags: BLE4.0
---
## 前言：
本文参考文献：  
1、[https://learn.adafruit.com/introduction-to-bluetooth-low-energy?view=all](https://learn.adafruit.com/introduction-to-bluetooth-low-energy?view=all)  
中文翻译：
[http://www.race604.com/gatt-profile-intro/](http://www.race604.com/gatt-profile-intro/)

2、[http://blog.csdn.net/jimoduwu/article/details/21604215](http://blog.csdn.net/jimoduwu/article/details/21604215)


## 一、工作原理

在说蓝牙通信之前，我们先了解一下蓝牙的工作原理。  
蓝牙规定每一对设备进行通讯时，必须一个为主角色，
另一为从角色，必须由主端进行查找，发起配对。连接成功，双方才可收发数据。  
一个具备蓝牙功能的设备，可以在两个角色间切换，
平时处于从模式，等待其它主设备来连接，需要时，转换为主模式，
向其它设备发起呼叫。  
一个蓝牙设备以主模式发起连接时，需要知道对方的蓝牙地址，配对密码等信息。

## 二、相关名词和概念

在介绍名词概念之前，我们应该知道，现在的BLE连接都是建立在   
GATT (Generic Attribute Profile) 协议之上。  
GATT 是一个在蓝牙连接之后，发送和接收很短的数据段的通用规范，
这些很短的数据段被称为属性（Attribute）。

### GAP

介绍 GATT 之前，需要了解 GAP (Generic Access Profile)，
它用来控制设备连接和广播。  
GAP 使一个设备可被其他设备发现，并决定了该设备是否可以或者怎样与其他设备进行交互。

GAP 给设备定义了若干角色，其中主要的两个是：
* **外围设备（Peripheral）：**一般是非常小或者简单的低功耗设备，用来提供数据，  
并能够连接到一个更加相对强大的中心设备。如蓝牙手环。
* **中心设备（Central）：**相对比较强大，用来连接其他外围设备。如手机。

### GATT

GATT 的全名是 Generic Attribute Profile（普通属性协议），  
它定义两个 BLE 设备通过叫做 Service 和 Characteristic 的东西进行通信。  
中心设备和外设需要双向通信的话，唯一的方式就是建立 GATT 连接。

##### GATT 结构

GATT 事务是建立在嵌套的Profiles、Services 和 Characteristics之上的，  
如图： 

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2825667-24ad1c9923858db6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

* **Profile：**  Profile 并不是实际存在于 BLE 外设上的，  
它只是一个被 Bluetooth SIG 或者外设设计者预先定义的 Service 的集合。

* **Service：**   Service 是把数据分成一个个独立逻辑项，  
它包含一个或者多个 Characteristic。每个 Service 有一个 UUID 唯一标识。

* **Characteristic：**   Characteristic在 GATT 事务中是最低级别的，  
是最小的逻辑数据单元，与 Service 类似，每个 Characteristic 也有一个UUID 唯一标识。

## 三、蓝牙通信
### 1、对应Android的API
在Android BLE的API中，对应于GATT结构的有3个文件：

* **BluetoothGatt** 
作为中央来使用和处理数据，通过device.connectGatt(this, false, mGattCallback) 得到。  
* **BluetoothGattServer**
作为周边来提供数据，通过BluetoothGatt.getService(uuid)获取指定的BluetoothGattServer。
* **BluetoothGattCharacteristic**
周边服务的一些特性，分为Read，Write，notification。

### 2、获取蓝牙连接状态

#### 2.1 BluetoothGattCallback

在上一篇中介绍了如何连接一个设备：

```
device.connectGatt(this, false, mGattCallback) 
```

其中有一个BluetoothGattCallback 的一个实例：mGattCallback。
BluetoothGattCallback是返回中央的状态和周边提供的数据的一个重要的抽象类。
看一下源码：

```

public abstract class BluetoothGattCallback {

    /**
     * 当GATT客户端已连接到GATT服务器或者从GATT服务器断开连接
     * 时回调。
     *
     * @param gatt 
     * @param status 连接或断开操作的状态。BluetoothGatt.GATT_SUCCESS表示操作成功
     *  
     * @param newState 返回新的连接状态。 如 BluetoothProfile.STATE_DISCONNECTED或
      * BluetoothProfile.STATE_CONNECTED
     */
    public void onConnectionStateChange(BluetoothGatt gatt, int status,
                                        int newState) {
    }

    /**
     * 当远程设备的远程服务列表，特征和描述符已被更新，即已发现新服务时，调用回调。
     *
     * @param gatt 
     * @param status BluetoothGatt.GATT_SUCCESS 远程设备的远程服务列表可被发现
     */
    public void onServicesDiscovered(BluetoothGatt gatt, int status) {
    }

    /**
     * Read特性的操作回调
     *
     * @param gatt 
     * @param characteristic 从相关的远程设备读取的特性。
     * @param status BluetoothGatt.GATT_SUCCESS 操作成功
     */
    public void onCharacteristicRead(BluetoothGatt gatt,   
                                     BluetoothGattCharacteristic characteristic,  
                                     int status) {
    }

    /**
     * write 特性的操作回调
     *
     * 如果在可靠的写事务正在进行时调用此回调，则characteristic的值表示远程设备报告的值。
     * 应用程序应该将此值与要写入的所需值进行比较。
     * 如果值不匹配，应用程序必须中止可靠的写事务。
     *
     * @param gatt 
     * @param characteristic 
     * @param status write 操作的结果
     *               {BluetoothGatt.GATT_SUCCESS}
     */
    public void onCharacteristicWrite(BluetoothGatt gatt,
                                      BluetoothGattCharacteristic characteristic,   
                                      int status) {
    }

    /**
     * notification 特性的结果
     *
     * @param gatt 
     * @param characteristic 由于远程通知事件而更新的特性。
     */
    public void onCharacteristicChanged(BluetoothGatt gatt,
                                        BluetoothGattCharacteristic characteristic) {
    }

    /**
     * 报告描述符读操作的结果的回调.
     *
     * @param gatt 
     * @param descriptor 从关联的远程设备读取的描述符。
     * @param status 如果读操作已成功完成 {BluetoothGatt.GATT_SUCCESS} 
     */
    public void onDescriptorRead(BluetoothGatt gatt, BluetoothGattDescriptor descriptor,
                                 int status) {
    }

    /**
     * 指示描述符写操作的结果。
     *
     * @param gatt 
     * @param descriptor 写入相关远程设备的描述符
     * @param status 如果操作成功，写入操作的结果为{BluetoothGatt＃GATT_SUCCESS}。
     */
    public void onDescriptorWrite(BluetoothGatt gatt, BluetoothGattDescriptor descriptor,
                                  int status) {
    }

    /**
     * 当可靠的写事务已完成时调用回调。
     *
     * @param gatt 
     * @param status 如果可靠的写事务已成功执行，则{BluetoothGatt＃GATT_SUCCESS}
     */
    public void onReliableWriteCompleted(BluetoothGatt gatt, int status) {
    }

    /**
     * 报告远程设备连接的RSSI。
     *
     * 此回调是响应{BluetoothGatt＃readRemoteRssi}函数而触发的。
     *
     * @param gatt 
     * @param rssi 远程设备的RSSI值
     * @param status 如果RSSI已成功读取 {BluetoothGatt＃GATT_SUCCESS}
     */
    public void onReadRemoteRssi(BluetoothGatt gatt, int rssi, int status) {
    }

    /**
     * 指示给定设备连接的MTU已更改。
     *
     * 此回调是响应{BluetoothGatt＃requestMtu}函数或响应连接事件而触发的。
     *
     * @param gatt 
     * @param mtu 新的MTU大小
     * @param status 如果MTU已成功更改{ BluetoothGatt＃GATT_SUCCESS}
     */
    public void onMtuChanged(BluetoothGatt gatt, int mtu, int status) {
    }
}
```


从上面的代码可以看出，当我们与一个外设建立连接时，
连接的状态以及连接成功之后与设备通信都会在BluetoothGattCallback中回调过来。

### 3、与设备的通信

#### 3.1、BluetoothGatt.discoverServices()

此方法在成功连接到远程设备时调用，不调用此方法，
无法与远程设备进行后续的通信。

```
@Override
    public void onConnectionStateChange(BluetoothGatt gatt, int status, int newState) {
        if (newState == BluetoothProfile.STATE_CONNECTED) {
            Log.i(TAG, "Connected to GATT server.");
            mBluetoothGatt.discoverServices();
        } else if (newState == BluetoothProfile.STATE_DISCONNECTED) {
            Log.i(TAG, "Disconnected from GATT server.");
        }
    }
```

但是这个方法是异步操作，在回调函数onServicesDiscovered中得到status，
通过判断status是否等于BluetoothGatt.GATT_SUCCESS来判断查找Service是否成功

```
@Override
    public void onServicesDiscovered(BluetoothGatt gatt, int status) {
        if (status == BluetoothGatt.GATT_SUCCESS) {
            //服务发现成功
        } else {
            //服务发现失败
        }
    }
```


只有当Service可被发现是即status为BluetoothGatt.GATT_SUCCESS时，
我们才能继续后续的操作。

#### 3.2、启用notification

如果设备主动发信息，可以通过notification的方式，
这种方式不用去轮询地读设备上的数据。

```
    BluetoothGattService service = bluetoothGatt.getService(serviceUuid);
	if (service == null) {
		return;
	}

	BluetoothGattCharacteristic characteristic =   
            service.getCharacteristic(characteristicUuid);
	if (characteristic == null) {
		return;
	}

	bluetoothGatt.setCharacteristicNotification(characteristic, true);//激活通知

	BluetoothGattDescriptor descriptor = characteristic.getDescriptor(descriptorUuid);
	descriptor.setValue(BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE);
	bluetoothGatt.writeDescriptor(descriptor);
```

如果notificaiton方式对于某个Characteristic是enable的，
那么当设备上的这个Characteristic改变时，
手机上的onCharacteristicChanged()回调就会被促发。


```

	@Override
	public void onCharacteristicChanged(BluetoothGatt gatt,
                                        BluetoothGattCharacteristic characteristic) {
	    super.onCharacteristicChanged(gatt, characteristic);

		byte[] notice = characteristic.getValue();

		onNotify(notice);
	}
```

#### 3.3、发送数据

```
BluetoothGattService service = bluetoothGatt.getService(serviceUuid);
	if (service == null) {
		return;
	}
	BluetoothGattCharacteristic characteristic =  
         service.getCharacteristic(characteristicUuid);
	if (characteristic == null) {
		return;
	}
	characteristic.setValue(data);
	bluetoothGatt.writeCharacteristic(characteristic);
}
```

#### 3.4、读取数据

```

	public void readData(UUID serviceUuid, UUID characteristicUuid) {
		BluetoothGattService service = bluetoothGatt.getService(serviceUuid);
		if (service == null) {
			return;
		}
		BluetoothGattCharacteristic characteristic =  
             service.getCharacteristic(characteristicUuid);
		if (characteristic == null) {
			return;
		}
		bluetoothGatt.readCharacteristic(characteristic);
	}
```

官方建议在进行read是如果有通知服务进行中，先关闭

```
bluetoothGatt.setCharacteristicNotification(characteristic, false);
```


读取的数据会在onCharacteristicRead()回调中返回


```
@Override
        public void onCharacteristicRead(BluetoothGatt gatt,
                                         BluetoothGattCharacteristic characteristic,
                                         int status) {
        	//读取到值，在这里读数据
            if (status == BluetoothGatt.GATT_SUCCESS) {
            }
        }
```

### 4、断开连接

```
public void disConnection() {
	if (bluetoothGatt == null) {
		return;
	}
	bluetoothGatt.disconnect();
}
```


#### 5、关闭Gatt

使用给定的BLE设备后，应用程序必须调用此方法以确保资源正确释放。


```
    /**
     *
     */
    public void close() {
        if (mBluetoothGatt == null) {
            return;
        }
        mBluetoothGatt.close();
        mBluetoothGatt = null;
    }
```

## 总结：
关于蓝牙BLE的开发流程，到这里就结束了，
从扫描、连接到与设备的收发数据，基本上也就这些东西。

最后加上GitHub上面的仓库：
[https://github.com/eson-yunfei/AndroidBle](https://github.com/eson-yunfei/AndroidBle) ；  
项目刚创建不久，还在完善，欢迎大家Fork。