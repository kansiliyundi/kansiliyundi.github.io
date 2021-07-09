---
title: ijkpalyer编译集成
tags:
  - iOS
categories: []
toc: false
date: 2020-03-10 12:24:55
---

> ijk安卓和iOS的编译过程大同小异,只要搞清楚配置文件的规则都是一样的，本文以iOS为例

## 拉代码
```
git clone https://github.com/Bilibili/ijkplayer.git ijkplayer-ios
# 进入代码所在文件夹
cd ijkplayer-ios
```

## 文件说明
### 关键文件说明：
```
# 初始化ios源码的脚本
ijkplayer-ios/init-ios.sh
# 初始化ios HTTPS支持的脚本
ijkplayer-ios/init-ios-openssl.sh

# 对应着ijkplayer在ios平台的源码
ijkplayer-ios/ios
./compile-ffmpeg.sh 编译ffmpeg脚本
./compile-openssl.sh 编译https支持脚本
# 执行以上脚本编译完成后的产物
ijkplayer-ios/ios/build


# 配置解码器配置文件所在的文件夹 
ijkplayer-ios/config
./module-default.sh 更多的编解码器/格式
./module-lite-hevc.sh 较少的编解码器/格式(包括hevc)
./module-lite.sh 较少的编解码器/格式(默认情况)
./module.sh 软连接以上三个其中一个 软连接方法 ln -s module-lite.sh module.sh

```
### 编辑配置说明：
```
#修改框架支持的arm指令集架构
ijkplayer-ios/init-ios.sh 中找到FF_ALL_ARCHS_IOS8_SDK字段,保留自己想要的架构
jkplayer-ios/init-ios-openssl.sh 同上
ijkplayer-ios/ios/compile-ffmpeg.sh 同上
ijkplayer-ios/ios/compile-openssl.sh 同上

# arm指令集说明
模拟器32位处理器测试需要i386架构，（iphone5,iphone5s以下的模拟器）
模拟器64位处理器测试需要x86_64架构，(iphone6以上的模拟器)
真机32位处理器需要armv7,或者armv7s架构，（iphone4真机/armv7,      ipnone5,iphone5s真机/armv7s）
真机64位处理器需要arm64架构。(iphone6,iphone6p以上的真机)
```

## 编译步骤
### 初始化源码
```
# 编辑配置
修改ijkplayer-ios/init-ios.sh支持的架构,修改方式看编辑配置一节
执行ijkplayer-ios/init-ios.sh
```
### HTTPS支持
```
如需支持HTTPS播放,需要执行ijkplayer-ios/init-ios-openssl.sh脚本初始化openssl源码
# 编辑配置
修改jkplayer-ios/init-ios-openssl.sh支持的架构,修改方式看编辑配置说明一节
   
# 在解码器配置文件中添加一行配置
export COMMON_FF_CFG_FLAGS="$COMMON_FF_CFG_FLAGS --enable-openssl"

执行ijkplayer-ios/init-ios-openssl.sh
```

### 编译
```
cd ios
# 编辑配置 修改支持的架构,修改方式看编辑配置说明一节
./compile-ffmpeg.sh 
./compile-openssl.sh

编译openssl, 如果不需要https可以跳过这一步
# 清理上次编译的产物
./compile-openssl.sh clean
# 编译openssl
./compile-openssl.sh all

编译ffmpeg
# 清理上次编译的产物
./compile-ffmpeg.sh clean
# 编译ffmpeg
./compile-ffmpeg.sh all 
```

### 添加支持库
```
如果执行了openssl编译,需要手动添加openssl的支持库
打开ijkplayer-ios/ios/IJKMediaPlayer/IJKMediaPlayer.xcodeproj
target->IJKMediaFramework->General->Frameworks and Libraries->添加->Add Other...
找到ijkplayer-ios/ios/build/universal/lib下的libcrypto.a 和libssl.a添加
```

### 合并动态库
```
修改Edit scheme -> Run -> Build Configuration为Release
分别选择模拟器和真机执行build
然后就可以在项目目录的Products文件夹下看到编译好的.framework库,show in finder
可以看到上一级有Release-iphoneos和Release-iphonesimulator两个文件夹，对应真机和模拟器
合并两个文件夹下的IJKMediaFramework.framework/IJKMediaFramework 为一个
lipo -create 真机framework路径 模拟器framework路径 -output 合并的文件路径
lipo -create Release-iphoneos/IJKMediaFramework.framework/IJKMediaFramework Release-iphonesimulator/IJKMediaFramework.framework/IJKMediaFramework -output IJKMediaFramework

执行后的IJKMediaFramework替换到原来的两个文件夹中即可
```

### 拖入项目
```
将Release-iphoneos/IJKMediaFramework.framework文件拖入自己项目的Framework文件夹，至此ijkplayer的编译以及导入全部完成
```