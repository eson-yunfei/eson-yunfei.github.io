---
layout: post
title: FFmpeg视频播放-SurfaceView
date: 2016-09-20 20:28:00 +0800
description:  # Add post description (optional)
tags: FFmpeg
---
之前已经把FFmpeg集成到项目里面了，剩下的就是做开发了，
做过安卓视频播放的都应该知道在播放的时候都有用到SurfaceView，
这里我们也采用这种方式。

## 一、定义Java层的调用接口
* 我们需要知道播放视频的网络地址或者是本地路径，并且希望这个地址是可以修改的，
    所以我们需要有一个参数去接收这个地址。

* 和系统一样，我们也需要传递一个Surface,在Jni中没有Surface这个类型，
    所以要用Object(JNI中除了基本的数据类型，其他的都用Object)。

* 开始播放，解码并进行播放视频

所以定义看三个方法，第一个有返回值的，返回小于1的初始化失败，
正常返回0。这个可以自行修改。

```
public class PlayerCore {

   /**
    * 设置播放路径
    *
    * @param path
    */
   public native int init(String path);

   /**
    * 设置播放渲染的surface
    *
    * @param surface
    */
   public native void setSurface(Object surface);

   /**
    * 开始播放
    */
   public native void start();
}
```

## 二、功能实现
### 1、个人理解的NDK
这个纯属个人理解，不对之处还请见谅，望指正。我把NDK开发分为Java、JNI和C/C++层。

* Java层。这个就不说了。
* JNI层：与Java层相对应的一层，是连接Java和C/C++的一个桥梁，这一层，必不可少。
* C/C++层：自己写的或者是别人写好的C/C++源码或者是库。

### 2、实现思路
   2.1、Java传递给JNI路径，JNI转换成C/C++的路径传递给解码器,即：
      
     String--->jstring--->const char *
     Java ---->JNI ------>C/C++
   
2.2设置Surface
    把Java的Surface传入JNI,由C/C++修改Surface的宽和高
           
    Surface-->>ANativeWindow
   
2.3、C/C++开始解码，并把解出来的视频帧，交由JNI层去显示到Surface上。

### 3、代码实现

围绕着这三个步骤，我们开始写代码。
JNI层要获取到视频的宽度和高度，还要拿到每一帧的图像去渲染，
所以就要有方法获取到视频的宽度和高度，如果采用直接读取的方式，有可以会读取失败，
所以，我采用了回调的方式。
先定义一个接口类(个人这么理解的,对于C/C++不是很通，先这么理解吧)。

```
class VideoCallBack {
public:
    //回调视频的宽度和高度
    virtual void onSizeChange(int width, int height);
    //回调解码出来的视频帧
    virtual void onDecoder(AVFrame *avFrame);
};
```

对应于C/C++层，我这里单独定义一个解码的类：FFDecoder对应于Java层。

```
class FFDecoder {
public:
    FFDecoder();
    int setMediaUri(const char *mediaUri);
    //在setSurface之后调用
    int setDecoderCallBack(VideoCallBack *videoCallBack);
    int startPlayMedia();
private:
    int findVideoInfo();
    static void *decoderFile(void *);
    static void setAVFrame(AVPacket *packet);
};
```

需要的方法都已定义好了，剩下就是实现了。开始已经说过，
我们要把Java传递过来的路径转换为FFDecoder能用的const char *mediaUri，
然后再传给FFDecoder，这里用到了JNI数据和C/C++的数据类型转换，
不会的自行百度或者谷歌。不要问我为什么，我也不是很懂。


```
ffDecoder = new FFDecoder();
const char *mediaUri = env->GetStringUTFChars(mediaPath, NULL);
int flag = ffDecoder->setMediaUri(mediaUri);
LOGE("mediaUri = %s", mediaUri);
LOGE("flag = %d", flag);
return flag;
```

FFDecoder拿到mediaUri之后，开始解码读取文件

```
 av_register_all();
 avcodec_register_all();
 avformat_network_init();
//前三句是注册解码相关的解码器，
//FFmpeg里面包含了很多的解码器，
 usleep(2 * 1000);

 int input = avformat_open_input(&avFormatContext, mediaPath, NULL, NULL);
 if (input < 0) {
     input = avformat_open_input(&avFormatContext, mediaPath, NULL, NULL);
 }
 if (input < 0) {
     LOGE(" open input error ,\n input ------->>%d", input);
     return -1;
 }
//设置最大缓存和最大读取时长
 avFormatContext->probesize = 4096;
 avFormatContext->max_analyze_duration = 1500;
 int streamInfo = avformat_find_stream_info(avFormatContext, NULL);
 if (streamInfo < 0) {
     LOGE(" find_stream error ,\n streamInfo ------->>%d", streamInfo);
     return -1;
 }
 // LOGE("streamInfo= %d",streamInfo);
 /*
  *输出文件的信息，也就是我们在使用ffmpeg时能够看到的文件详细信息，
  *第二个参数指定输出哪条流的信息，-1代表ffmpeg自己选择。最后一个参数用于
  *指定dump的是不是输出文件，我们的dump是输入文件，因此一定要为0
  */
 av_dump_format(avFormatContext, -1, mediaPath, 0);
 avPacket = av_packet_alloc();
// avPacket = (AVPacket *)
 av_malloc(sizeof(AVPacket));
 findVideoResult = findVideoInfo();
 if (findVideoResult < 0) {
     return -1;
 }
 return 0;
```

ps:  
这里说明一下为什么我在开始的时候，代码里面会有一个延时和读取两次，
因为我在做实际项目中，有一个切换视频分辨率的功能，在切换的时候，
原始的数据流断开了，我这边需要重新连接，在重新连接的时候，
如果我打开的太快，视频流地址还没有开启，
所以我就加了一个延时和重新读取。如果是本地视频播放，可忽略。

下面是findVideoInfo()的内容，
最主要的就是获取videoStreamIndex、video_width和video_height。

```
int FFDecoder::findVideoInfo() {
  //视频流标志，如果是-1说明没有找到视频相关信息
    videoStreamIndex = -1;
    for (int i = 0; i < avFormatContext->nb_streams; i++) {
        if (avFormatContext->streams[i]->codec->codec_type == AVMEDIA_TYPE_VIDEO
            && videoStreamIndex < 0) {
            videoStreamIndex = i;
            break;
        }
    }
    if (videoStreamIndex < 0) {
        LOGE("Didn't find a video stream ");
        return -1;
    }
    // LOGE("videoStreamIndex --->>%d", videoStreamIndex);
    videoStream = avFormatContext->streams[videoStreamIndex];
    // Get a pointer to the codec context for the video stream
    videoCodecContext = videoStream->codec;
    // Find the decoder for the video stream
    videoCodec = avcodec_find_decoder(videoCodecContext->codec_id);
    if (videoCodec == NULL) {
        LOGE("videoAvCodec not found.");
        return -1;
    }
    if (avcodec_open2(videoCodecContext, videoCodec, NULL) < 0) {
        LOGE("Could not open videoCodecContext.");
        return -1;
    }
    //视频帧率
    float rate = (float) av_q2d(videoStream->r_frame_rate);
    LOGE("rate--------->>%f", rate);
    //视频的宽和高
    video_width = videoCodecContext->width;
    video_height = videoCodecContext->height;
    LOGE("video_width--------->>%d", video_width);
    LOGE("video_height--------->>%d", video_height);
    if (video_width == 0 || video_height == 0) {
        return -1;
    }
    return 1;
}
```

然后是设置我们的Surface,并且设置视频的回调


```
    mANativeWindow = NULL;
    // 获取native window
    mANativeWindow = ANativeWindow_fromSurface(env, surface);  
    if (mANativeWindow == NULL) {
        LOGE("ANativeWindow_fromSurface error");
        return;
    }
    ffDecoder->setDecoderCallBack(new VideoCallBack());
```

FFDecoder接收到回调VideoCallBack的指针之后，
设置视频的宽和高并初始化视频的渲染格式，这里采用的是RGBA。


```
int FFDecoder::setDecoderCallBack(VideoCallBack *videoCallBack) {
    mVideoCallBack = videoCallBack;
    mVideoCallBack->onSizeChange(video_width, video_height);
 
    pFrame = av_frame_alloc();
    // 用于渲染//
    pFrameRGBA = av_frame_alloc();
    // Determine required buffer size and allocate buffer

    int numBytes = av_image_get_buffer_size(AV_PIX_FMT_YUV420P,
                                            videoCodecContext->width,
                                            videoCodecContext->height,
                                            1);
    uint8_t *buffer = (uint8_t *) av_malloc(numBytes * sizeof(uint8_t));
    av_image_fill_arrays(pFrameRGBA->data,pFrameRGBA->linesize,
                         buffer,AV_PIX_FMT_YUV420P,
                         videoCodecContext->width,
                         videoCodecContext->height, 1);
    // 由于解码出来的帧格式不是RGBA的,在渲染之前需要进行格式转换//
    sws_ctx = sws_getContext(videoCodecContext->width,//
                             videoCodecContext->height,//
                             videoCodecContext->pix_fmt,//
                             videoCodecContext->width,//
                             videoCodecContext->height,//
                             AV_PIX_FMT_YUV420P,//
                             SWS_FAST_BILINEAR,//
                             NULL,//
                             NULL,//
                             NULL);
}
```

拿到视频的宽度和高度之后，进行设置我们的mANativeWindow，
并且设置为WINDOW_FORMAT_RGBA_8888。

```
void VideoCallBack::onSizeChange(int width, int height) {
    w_width = width;
    w_height = height;
//    LOGE("w_width--------->>%d", w_width);
//    LOGE("w_height--------->>%d", w_height);
    if (w_width == 0 || w_height == 0) {
        return;
    }
    // 设置native window的buffer大小,可自动拉伸//
    ANativeWindow_setBuffersGeometry(mANativeWindow, w_width, w_height,//
                                     WINDOW_FORMAT_RGBA_8888);
}
```

接下来就是开始播放，因为我们要去不断的读取视频里面的AVPacket,
并且要从AVPacket里面获取的原始的AVFrame，所以这些我放在了线程里面去操作。


```
int FFDecoder::startPlayMedia() {
    //开启文件解码线程
    pthread_create(&decoderThread, NULL, decoderFile, NULL);
}
```

startPlayMedia只做一件事情，就是开启解码的线程，
真正要做事的实在decoderFile这个指针函数里面


```
void* FFDecoder::decoderFile(void *) {
    while (true){
   
      //usleep(20 * 1000);//中间的延时,如果不加这一句，
                          //播放本地视频的时候就如同视频快进一样，每一帧图片一闪而过
        int readFrame = av_read_frame(avFormatContext, avPacket);
        if (readFrame < 0) {
            // LOGE(" readFrame is < 0 ------------->%d", readFrame);
            break;
        }
        int packetStreamIndex = avPacket->stream_index;
            if (packetStreamIndex == videoStreamIndex) {
                setAVFrame(avPacket);
            }
    }
}

/**
 *
 */
void FFDecoder::setAVFrame(AVPacket *packet) {
    int gotFrame = -1;
    int line = avcodec_decode_video2(videoCodecContext, pFrame, &gotFrame, packet);
    if (line < 0) {
        LOGE("line----------->>%d", line);
        av_free_packet(packet);
        return;
    }
    if (gotFrame < 0) {
        LOGE("gotFrame----------->>%d", gotFrame);
        av_free_packet(packet);
        return;
    }
    int errflag = pFrame->decode_error_flags;
    if (errflag == 1) {
        av_free_packet(packet);
        return;
    }
    // 格式转换//
    sws_scale(sws_ctx, (uint8_t const *const *) pFrame->data,//
              pFrame->linesize, 0, videoCodecContext->height,//
              pFrameRGBA->data, pFrameRGBA->linesize);
    //回调解码出视频帧
    mVideoCallBack->onDecoder(pFrameRGBA);
    av_free_packet(packet);
}
```

拿到视频帧之后，剩下的就是如果渲染到Surface上。

```
void VideoCallBack::onDecoder(AVFrame *avFrame) {
    if (w_width == 0 || w_height == 0) {
        return;
    }
    ANativeWindow_lock(mANativeWindow, &windowBuffer, 0);//
    
    if (windowBuffer.stride == 0) {
        LOGE("surface 创建失败");//
        return;//
    }
    // 获取stride//
    uint8_t *dst = (uint8_t *) windowBuffer.bits;//
    if (dstStride == 0) {//
        dstStride = windowBuffer.stride * 4;//
    }
//    // LOGE("dstStride------>>>%d", dstStride);
    uint8_t *src = avFrame->data[0];
    int srcStride = avFrame->linesize[0];
    // LOGE("srcStride------>>>%d", srcStride);
    // 由于window的stride和帧的stride不同,因此需要逐行复制
    int h;//
    for (h = 0; h < w_height; h++) {//
        memcpy(dst + h * dstStride, src + h * srcStride, srcStride);
    }

    ANativeWindow_unlockAndPost(mANativeWindow);
}
```

到这里，核心的代码已经写完了，剩下的就是去编译，
然后在Java里面去调用。就可以去播放视频了。
至于音频和其他的一些功能，有时间在写吧。

------------------------------------

参考链接：  
 [Android+FFmpeg+ANativeWindow视频解码播放](http://blog.csdn.net/glouds/article/details/50937266)