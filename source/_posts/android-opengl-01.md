---
title: Android OpenGL - 01.OpenGL介绍及项目搭建
date: 2020-03-27 00:48:45
tags: 
- android
- opengl
cover: https://blog-1256162814.cos.ap-nanjing.myqcloud.com/opengl/opengl_banner.jpg
---

> 从今天开始，我会学习Android OpenGL相关知识，也会将学习到的知识进行总结并整理到这里。

# OpenGL介绍及开发环境搭建

![](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/opengl/OpenGL%E4%BB%8B%E7%BB%8D%E5%8F%8A%E9%A1%B9%E7%9B%AE%E6%90%AD%E5%BB%BA.PNG)
<!-- more -->

## OpenGL 简单介绍

OpenGL（OpenGraphicsLibrary）定义了一个跨编程语言、跨平台编程的专业图形程序接口。可用于二维或三维图像的处理与渲染，它是一个功能强大、调用方便的底层图形库。对于嵌入式的设备，其提供了OpenGLES（OpenGLforEmbeddedSystems）版本，该版本是针对手机、Pad等嵌入式设备而设计   的，是OpenGL的一个子集。到目前为止，OpenGL已经经历过很多版本的迭代与更新，最新版本为3.0，而使用最广泛的还是OpenGLES2.0版本。

## EGL

EGL 是在不同平台上提供 OpenGL 上下文及窗口管理能力的具体实现，在 Android 中使用的是 EGL 在 iOS 是使用的是EAGL。

## EGL 线程？

OpenGLES 的渲染工作都是需要在指定线程中完成，在该线程中，需要绑定OpenGLES 的上下文及 Surface。绑定完成后即可以进行 OpenGLES 的渲染工作。此时的线程就叫做 EGL 线程。

## EGL、EGL 线程和 Surface之间的关系

![](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/opengl/opengl_es_surface.png)


# 创建c++项目并导入OpenGL

## 编译环境

需要创建支持 c++ 的 Android 项目，并且下载LLDB、CMAKE 和 NDK。
如果编译后提示找不到cmake，需要在项目根目录的local.properties中添加如下路径：

```gradle
cmake.dir=C\:\\Users\\zqlxt\\AppData\\Local\\Android\\Sdk\\cmake\\3.10.2.4988404
```

## 导入OpenGL相关库

在 CMakeLists.txt 中添加如下依赖：

```
find_library( # Sets the name of the path variable.
              log-lib

              # Specifies the name of the NDK library that
              # you want CMake to locate.
              log
              EGL  # egl依赖
              GLESv2  # opengles 依赖
              android)  # android ndk依赖
```