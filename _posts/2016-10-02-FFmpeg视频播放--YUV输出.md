---
layout: post
title: FFmpeg视频播放--YUV输出
date: 2016-09-20 20:28:00 +0800
description:  # Add post description (optional)
tags: FFmpeg
---

之前用的Android SurfaceView播放视频是采用的把surface丢到JNI层，
在里面更新视图,这种方式只能渲染 AV_PIX_FMT_RGBA 的格式。
但是，由于FFmpeg解码出来的格式默认是YUV的数据，
所以解码出来之后我们要转换成为RGBA的，
这个转换的操作是很耗时和耗性能的，所以就需要直接使用YUV数据。

## 1、了解YUV数据来源
首先我们要知道，不管是YUV 还是RGBA或者其他的格式，
每一帧数据都是存储在AVFrame里面的，
那么我们就要先了解一下AVFrame。
关于AVFrame，网上有很多的介绍，我这里也不多说，
这里给出雷神关于AVFrame的讲解：[FFMPEG结构体分析：AVFrame](http://blog.csdn.net/leixiaohua1020/article/details/14214577) 
了解过AVFrame之后，我们知道有两个很重要的数组：
```
/**
 * pointer to the picture/channel planes.
 * 图像数据
 * This might be different from the first allocated byte
 */
uint8_t *data[AV_NUM_DATA_POINTERS];
/**
 * For video, size in bytes of each picture line.
 * 对于视频，每一帧图象一行的字节大小。
 * For audio, size in bytes of each plane.
 */

int linesize[AV_NUM_DATA_POINTERS];
```

雷神说：
> uint8_t *data[AV_NUM_DATA_POINTERS]：
解码后原始数据（对视频来说是YUV，RGB，对音频来说是PCM）。    
 int linesize[AV_NUM_DATA_POINTERS]：data中“一行”数据的大小。
注意：未必等于图像的宽，一般大于图像的宽

所以很清楚，我们的数据就是从这两个数据里面来获取YUV数据。
如何在C/C++层获取YUV数据，
参考雷神的另外一篇文章：  
[FFMPEG 实现 YUV，RGB各种图像原始数据之间的转换（swscale）](http://blog.csdn.net/leixiaohua1020/article/details/14215391)

我这里只贴出关键代码，具体的去看雷神的帖子

```
//YUV420P 
fwrite(pFrameYUV->data[0],(pCodecCtx->width)*(pCodecCtx->height),1,output); 
fwrite(pFrameYUV->data[1],(pCodecCtx->width)*(pCodecCtx->height)/4,1,output); 
fwrite(pFrameYUV->data[2],(pCodecCtx->width)*(pCodecCtx->height)/4,1,output); 
```

根据上面的代码我们知道：  
Y的数据的长度=视频的原始宽(pCodecCtx->width)  
× 视频的原始高度(pCodecCtx->height)  
u的数据的长度 = v = y/4

## 2、编写代码

知道这些之后，开始写我们今天的代码

### 2.1、定义方法：

```
/**
 * JNI 回调视频的宽度和高度
 */  
private void setMediaSize(int width, int height) { }
/**
 * JNI 回调每一帧的YUV数据
 */
private void onDecoder(byte[] yData, byte[] uData, byte[] vData) {}
```

注：这里主要讲YUV的数据的回调，
所以以下代码无关乎 setMediaSize(int width, int height)。

### 2.2、找到Java类里面的方法：

```
//定义Java类的包名和类名
const char *J_CLASS_NAME = "com/eson/player/MyFPlayerCore";
jclass playerCore; 
//java方法
jmethodID onDecoder;

playerCore =jniEnv->FindClass(J_CLASS_NAME);
onDecoder = jniEnv->GetMethodID(playerCore, "onDecoder", "([B[B[B)V");
```

### 2.3、获取YUV数据

我们还是用之前写的onDecoder(AVFrame *avFrame)这个方法，只需要修改一下就行。

```
JNIEnv *jniEnv;
jbyteArray yArray;
jbyteArray uArray;
jbyteArray vArray;
int length = 0;
unsigned char *ydata;
unsigned char *udata;
unsigned char *vdata;

void VideoCallBack::onDecoder(AVFrame *avFrame) {
//    LOGE("onDecoder (AVFrame)");
    if (w_width == 0 || w_height == 0) {
        return;
    }
    if (!avFrame) {
        return;
    }
    //这里只是获取到数据的指针
    ydata = avFrame->data[0];
    udata = avFrame->data[1];
    vdata = avFrame->data[2];

  //刚开始读数据前几帧数据有空数据，不知道为什么
    if (ydata == NULL || udata == NULL || vdata == NULL) {
        return;
    }

  //数据的长度，即Java byte[] 的长度
    if (length == 0) {
        length = w_width * w_height;
    }
  
    if (jniEnv == NULL) {
        jniEnv = callJavaUtil->getCurrentJNIEnv();
        LOGE(" got new jnienv");
    }
    //只初始化一次长度
    if (yArray == NULL) {
        yArray = jniEnv->NewByteArray(length);
        LOGE(" got new yArray");
    }
  
    jniEnv->SetByteArrayRegion(yArray, 0, length, (jbyte *) ydata);
    if (uArray == NULL) {
        uArray = jniEnv->NewByteArray(length / 4);
        LOGE(" got new uArray");
    }
   
    jniEnv->SetByteArrayRegion(uArray, 0, length / 4, (jbyte *) udata);
    if (vArray == NULL) {
        vArray = jniEnv->NewByteArray(length / 4);
        LOGE(" got new vArray");
    }
 
    jniEnv->SetByteArrayRegion(vArray, 0, length / 4, (jbyte *) vdata);
    //回调
    callJavaUtil->callOnDecoder(jniEnv, yArray, uArray, vArray);
}
```


我们再打印一下linesize的长度，看一下与视频宽度(w_height)的关系


```
LOGE("avFrame->linesize[0] ----->>>%d",avFrame->linesize[0]);
LOGE("avFrame->linesize[1] ----->>>%d",avFrame->linesize[1]);
LOGE("avFrame->linesize[2] ----->>>%d",avFrame->linesize[2]);
```


经过几个视频的测试会发现，  
avFrame->linesize[0] 
始终是avFrame->linesize[1]和avFrame->linesize[2]的2倍，
而avFrame->linesize[0] 和w_height是相等的。
可雷神说是不总是相等的，那怎么办?

### 2.4、完善

后来经过查资料，在这里找到了解决方法：
[ffmpeg从AVFrame取出yuv数据到保存到char*中](http://www.cnblogs.com/1024Planet/p/5803382.html)
参照他的方法我对代码进行了修改：

```
JNIEnv *jniEnv;
jbyteArray yArray;
jbyteArray uArray;
jbyteArray vArray;
int length = 0;
unsigned char *ydata;
unsigned char *udata;
unsigned char *vdata;

//新的yuv数据
uint8_t *newY = NULL;
uint8_t *newU = NULL;
uint8_t *newV = NULL;

void VideoCallBack::onDecoder(AVFrame *avFrame) {
//    LOGE("onDecoder (AVFrame)");
    if (w_width == 0 || w_height == 0) {
        return;
    }
    if (!avFrame) {
        return;
    }
    ydata = avFrame->data[0];
    udata = avFrame->data[1];
    vdata = avFrame->data[2];
    if (ydata == NULL || udata == NULL || vdata == NULL) {
        return;
    }

    //长度不变，不改变原始图像的宽高
    if (length == 0) {
        length = w_width * w_height;
    }
    //重新申请一个与所需相同的内存
    if (newY == NULL) {
        newY = (uint8_t *) av_malloc(length * sizeof(uint8_t));
    }
    if (newU == NULL) {
        newU = (uint8_t *) av_malloc(length / 4 * sizeof(uint8_t));
    }
    if (newV == NULL) {
        newV = (uint8_t *) av_malloc(length / 4 * sizeof(uint8_t));
    }

    //把原始数据复制到申请的内存里面
    for (int i = 0; i < w_height; i++) {
        memcpy(newY + w_width * i,
               ydata + avFrame->linesize[0] * i,
               w_width);
    }
    for (int j = 0; j < w_height / 2; j++) {
        memcpy(newU + w_width / 2 * j,
               udata + avFrame->linesize[1] * j,
               w_width / 2);
    }
    for (int k = 0; k < w_height / 2; k++) {
        memcpy(newV + w_width / 2 * k,
               vdata + avFrame->linesize[2] * k,
               w_width / 2);
    }
    if (jniEnv == NULL) {
        jniEnv = callJavaUtil->getCurrentJNIEnv();
    }
    //只初始化一次长度
    if (yArray == NULL) {
        yArray = jniEnv->NewByteArray(length);
    }
   
    //把新的数据放到byte[]里面
    jniEnv->SetByteArrayRegion(yArray, 0, length, (jbyte *) newY);
    if (uArray == NULL) {
        uArray = jniEnv->NewByteArray(length / 4);
    }
    if (uArray == NULL){
        return;
    }
    jniEnv->SetByteArrayRegion(uArray, 0, length / 4, (jbyte *) newU);
    if (vArray == NULL) {
        vArray = jniEnv->NewByteArray(length / 4);
    }
    if (vArray == NULL){
        return;
    }
    jniEnv->SetByteArrayRegion(vArray, 0, length / 4, (jbyte *) newV);
    //回调
    callJavaUtil->callOnDecoder(jniEnv, yArray, uArray, vArray);
}
```


到这里，AVFrame里面的YUV数据就以byte[]的方式传递到了Java层了。

最后说一下，由于之前公司离职，所以关于视频解码这一块的文章估计会停更，
也或许不会，看情况吧。