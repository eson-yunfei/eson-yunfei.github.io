---
layout: post
title: Android BLE4.0(基本知识)
date: 2016-12-25 13:36:00 +0800
description:  # Add post description (optional)
tags: BLE4.0
---
## 写在前面的话

好长一段时间没有写文章了，因为一些个人原因吧，这里就不说了。
最近接了一个关于蓝牙的项目，虽然之前也有搞过相关的开发，
当时也封装了一套简单的Api，但是并没有仔细研究过这一块，
正好趁这次机会再好好的研究一些，顺便记录一下。
关于各种理论知识其实我是不想多说的，因为网上已经有很多了，
但是为了能够让自己在加深一些还是在这里啰嗦一下吧。
## 理论知识

### 什么是蓝牙？

**蓝牙**是一种无线技术标准，可实现固定设备、
移动设备和楼宇个人域网之间的短距离数据交换，
最初由电信巨头[爱立信公司](http://baike.baidu.com/view/1066191.htm)于1994年创制，
如今蓝牙由蓝牙技术联盟（Bluetooth Special Interest Group，简称SIG）管理。

### 蓝牙历史

蓝牙发展至今经历了8个版本的更新。1.1、1.2、2.0、2.1、3.0、4.0、4.1、4.2。
在1.x~3.0之间称为传统蓝牙，4.0开始称为低功耗蓝牙也就是蓝牙ble。
这里我们主要说的就是4.x的低功耗蓝牙。

### 低功耗蓝牙

低功耗蓝牙是由蓝牙技术联盟在2010年6月30号推出的。
- 低功耗蓝牙较传统蓝牙，
具有**传输速度更快，覆盖范围更广，安全性更高，延迟更短，耗电极低**等优点。
这也是为什么近年来智能穿戴的东西越来越多，越来越火。
- 传统蓝牙与低功耗蓝牙通信方式也有所不同，传统的一般通过socket方式，而低功耗蓝牙是通过Gatt协议来实现。

就先说这么多吧。

### 部分概念

- **Generic Attribute Profile (GATT)**  
通过BLE连接，读写属性类小数据的Profile通用规范。现在所有的BLE应用Profile都是基于GATT的。
 
- **Attribute Protocol (ATT)**  
GATT是基于ATT Protocol的。ATT针对BLE设备做了专门的优化，
具体就是在传输过程中使用尽量少的数据。每个属性都有一个唯一的UUID，
属性将以characteristics and services的形式传输。
 
- **Characteristic**  
Characteristic 可以理解为一个数据类型,
它包括一个单一变量和0-n个用来描述characteristic变量的descriptor，类似于类
 
- **Descriptor**  
Descriptor用来描述characteristic变量的属性。例如，一个descriptor可以规定一个可读的描述，  
或者一个characteristic变量可接受的范围，或者一个characteristic变量特定的测量单位
 
- **Service**  
Characteristic的集合。

## 小结
这里说小结有点不合适，嘿嘿嘿。。。。本来是想接着往下面写的，
但是如果接着写篇幅会有点长，所以打算分开来写。关于概念性的东西就先写这么多。
后续记录蓝牙在Android中的运用。