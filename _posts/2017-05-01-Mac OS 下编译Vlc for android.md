---
layout: post
title: Mac OS 下编译Vlc for android
date: 2017-05-01 21:30:00 +0800
description:  # Add post description (optional)
tags: VLC
---

## VLC for Android简介

关于VLC for Android，做过音视频相关的开发者应该都听说过他的大名，
[官方(VideoLAN)](http://www.videolan.org/vlc/download-android.html)是这么介绍的：

> 
**VLC for Android**  
>  
VLC media player is a `free` and `open source` cross-platform multimedia 
player that plays most multimedia files as well as discs, devices, and network streaming protocols.   
>  
VLC for Android can play any video and audio files, 
as well as network streams and DVD ISOs, like the desktop version of VLC.  
>  
VLC for Android is a full audio player, 
with a complete database, an equalizer and filters,
playing all weird audio formats.

翻译成中文呢。。。咳咳，不好意思，英文水平有限就不翻译了，
大概就是说：*VLC for Android可以播放任何视频和音频文件，
并且是一个完整的音频播放器，可以播放所有的音频格式。*

听起来是不是很吊炸天啊，嗯嗯，而事实也确实是这样的。
那么我们就来编译一下VLC for Android的源码。

## 为啥要写这篇文章

其实已经有很多人编译过VLC for Android的源码，大家去网络上搜索一下，
也可以搜索到很多相关的文章。当然，我也一样去搜索。但是，But，
当我按照他们的编译步骤去做的时候，呵呵。。不说了。
可能是我的电脑有点特殊，就是死活编译不过去。无奈，不看你们的了，
我自己玩吧。所以就记录一下这个历程吧。

## 正题

### 1、准备工作

#### 1.1、获取源码

既然我们要编译源码，那么首先我们要先获取源码：

```
git clone https://code.videolan.org/videolan/vlc-android.git
```

#### 1.2、编译环境配置

获取到源码之后呢，就要配置编译环境。这里先给出
[官方的编译指南](https://wiki.videolan.org/AndroidCompile#Get_VLC_Source)。
Android的SDK、NDK 环境这个自不必说，做Android开发的应该都已经配置好了，
这个地方提一点，就是这个环境变量的名字，
最好是和官方的保持一致：`ANDROID_SDK`，`ANDROID_NDK `。
至于路径，你开心就好。除了SDK、NDK 官方还给出了以下一堆的编译工具：

```
sudo apt-get install automake ant autopoint cmake build-essential libtool \
     patch pkg-config protobuf-compiler ragel subversion unzip git \
    openjdk-8-jre openjdk-8-jdk
```

其中`sudo apt-get install`是`Linux`下的软件安装命令，
可以不看。其实啊，就我个人来看，这些软件并没有全部用上。
我在编译的过程中，是遇到哪一个提示“找不到指令”是才去安装的，
并没有一下子全部安装。而且，就算你一下子去安装所有，我可以保证的是，
有几个是找不到安装资源的。别问我为什么知道。我这里给出我安装的几个：

```
automake 、ant  、cmake、libtool
```

有人会问，`git` 、`subversion `为什么不安装，
其实我在想，你为什么会问这个问题。难道你这些常用的代码同步软件你不用的吗？
哦，忘记一个关键问题，
就是在Mac下命令行安装软件用的是 [Homebrew](https://brew.sh/)。

### 2、正式编译

#### 2.1、*sh compile.sh*

使用上面的 `git clone` 命令，如果不做更正，
会在终端当前的文件夹(终端查看当前文件夹，直接输入：`pwd`)
下生成一个`vlc-android`的文件夹，进入这个文件夹，
会看到两个`.sh`文件：

![sh.png](http://upload-images.jianshu.io/upload_images/2825667-1c255faec9b9c57f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中`compile-libvlc.sh`只编译vlc 库文件，
不编译Vlc for android，如果想编译Vlc for android就要用`compile.sh`。
编译平台可以参考
[官方的编译指南](https://wiki.videolan.org/AndroidCompile#Get_VLC_Source)。


```
sh compile.sh -a <ABI>
//也可以不指定，直接使用：sh compile.sh
//注意：指定的是说明使用默认的编译平台：ARMv7
```

我编译的时候，未指定编译平台，直接使用默认的。

#### 2.2、资源下载

执行上面的命令之后，会检测你的环境和资源是否完整，
如果不完整会下载相关的资源，至于会下载多久，视你的网络状况而定。

#### 2.3、编译

这一步是最关键的一步，如果你运气好的话，看到如下字样的时候，

![good luck.png](http://upload-images.jianshu.io/upload_images/2825667-671594db0bf93dbb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下面的就不用看了，如果运气不好，希望接下来的文字对你有所帮助。
如果没有帮助，也没有办法，毕竟每个人的电脑环境都不一样，
遇到的问题也不一样，谁知道你电脑里面有什么不干净的东西，
反正我的电脑在一个月前重装了系统。

### 3、编译过程中的问题

其实，上面说了那么多，只有到这里，才是最关键的，
因为上面说的，和别人说的都一样，不一样的就是大家遇到的问题不一样。
不然，我这一篇文章也就没有必要去写了。

好多人都曾经尝试过编译VLC，但是当遇到了问题，
或百度、或谷歌、或者其他的搜索，结果却没有得到一个有实用价值的答案，
最后不了了之。这就扼杀了许多有志码农学习VLC的愿望。
还记得1年前我也是其中一员，如今，又走上了这条不归路。。

#### 3.1、相关资源下载问题。

其实这不是个问题，翻下墙就好了。不过，这次编译，好像并没有翻墙。。。
* **early EOF**
关于这个问题，多试几次就好了
```
error: RPC failed; curl 18 transfer closed with outstanding read data remaining
fatal: The remote end hung up unexpectedly
fatal: early EOF
fatal: index-pack failed
```

	没必要去百度`git early EOF`的解决方案。

* **contribs: make fetch failed**
同上，多试几次。


#### 3.2、编译问题
* **编译FFmpeg 是出现：{standard input}:146: Error: unknown register alias 'GP'**  
解决方案：  
在vlc-android/vlc/contrib/src/ffmpeg/rules.mak文件中，添加如下代码：
```
FFMPEGCONF = \
	--cc="$(CC)" \
	--pkg-config="$(PKG_CONFIG)" \
	--disable-doc \
	--disable-asm \          //添加此行代码
	--disable-encoder=vorbis \
	--disable-decoder=opus \
	--enable-libgsm \
	--enable-libopenjpeg \
	--disable-debug \
	--disable-avdevice \
	--disable-devices \
	--disable-avfilter \
	--disable-filters \
	--disable-protocol=concat \
	--disable-bsfs \
	--disable-bzlib \
	--disable-avresample
```

*  **medialibrary 编译错误**
```
  CXX      src/libmedialibrary_la-Album.lo
  CXX      src/libmedialibrary_la-AlbumTrack.lo
  CXX      src/libmedialibrary_la-Artist.lo
  CXX      src/libmedialibrary_la-AudioTrack.lo
clang38++: error: no such file or directory: '@includedirs@'
clang38++: error: no such file or directory: '@includedirs@'
clang38++: error: no such file or directory: '@includedirs@'
clang38++: error: no such file or directory: '@includedirs@'
make[1]: *** [src/libmedialibrary_la-AlbumTrack.lo] Error 1
make[1]: *** Waiting for unfinished jobs....
make[1]: *** [src/libmedialibrary_la-Album.lo] Error 1
make[1]: *** [src/libmedialibrary_la-AudioTrack.lo] Error 1
make[1]: *** [src/libmedialibrary_la-Artist.lo] Error 1
``` 
解决方案：  
打开`/vlc-android/medialibrary/medialibrary/build-android-armeabi-v7a/`的Makefile 文件，  

修改第624行：
```
//修改之前
VLC_CFLAGS = @includedirs@ 
//修改之后

vlc_path = /vlc-android/vlc  //这里指向你的vlc-android/vlc的路径，全路径
vlc_include = ${vlc_path}/include
vlc_version_include = ${vlc_path}/build-android-arm-linux-androideabi/include
VLC_CFLAGS =  -I${vlc_include}  -I${vlc_version_include}
```

差不多就这些问题吧，感觉这次编译要比之前编译的时候，容易了很多，如果有问题，欢迎大家留言。