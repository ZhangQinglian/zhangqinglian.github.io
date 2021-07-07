---
title: Android OpenGL ES 入门 3 - OpenGL ES 绘制三角形
date: 2021-01-27 22:32:51
tags:
  - android
  - opengl
---
在前面几篇文章中我们已经初始化了GLSurfaceView 和 EGL，并且使用 OpenGLES（下文简称GLES）渲染出了红色画面。

这一节中我们将在红色上继续画上三角形。

在着手画三角形之前，我们先要学习一点点关于 GLES 的预备知识，非常简单。
1. GLES 底层是由 c 语言实现，所以没有面向对象特性，它的接口及工作原理更接近面向过程，且其内部就是一个巨大的状态机。例如前面画的红色我们做了两步操作，第一步`GLES32.glClearColor`，设置清屏颜色，第二步`GLES32.glClear`进行清屏。
总结一下就是 **GLES 的渲染一般经历两步：1. 状态设置，2. 状态使用。** 牢记这一点，对后面的学习会有很大的帮助。

2. GLES 的渲染工作是在 GPU 中完成，所以我们的模型数据都需要由 CPU 转交给 GPU。
    
3. GLES 中的坐标是标准化设备坐标（Normalized Device Coordinates, NDC），标准化设备坐标是一个x、y和z值在-1.0到1.0的一小段空间。任何落在范围外的坐标都会被丢弃/裁剪，不会显示在你的屏幕上。如下图：
![](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/opengl/ndc.png)
所以如果我们在手机上直接画（0,0.5）、（0.5,0.5）和（-0.5,0.5）这三个点的时候，得到的就未必是上图中的等边三角形了，因为我们屏幕并不是一个正方形，某个方向是会被拉伸的。具体怎么解决这个问题，在以后的章节中会讲到。

差不多就是以上三点，把他们都牢记在心，对入门 GLES 很重要。

<!--more-->
# 绘制三角形
基于上面说道的三点，我们来考虑下如果要绘制三角形，需要做哪些事情？
1. 需要有三角形三个点坐标。
2. 需要知道三角形的颜色。
3. 需要把三角形的坐标和颜色告诉 GPU。
4. 让 GPU 绘制三角形。

差不多就是这么多了，下面我们开始写代码。

## 准备三角形坐标和颜色
这个很简单，直接看代码：
```java
    private static final float TRIANGLE_COORDS[] = {
            0.5f, 0.5f,   // 0 top
            -0.5f, 0.5f,   // 1 bottom left
            0f, -0.5f    // 2 bottom right
    };

    private static final float TRIANGLE_COLOR[] = {0.1F, 0.9F, 0.4F, 1.0F};
```
将数组坐标和颜色值放入数组中。

## 把三角形坐标和颜色告诉GPU
把三角形坐标和颜色告诉GPU就没那么简单了，需要使用到 GLSL（着色器语言）。
在介绍GLSL前首先介绍下GLES的渲染流程，俗称渲染管线。
![](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/opengl/pipeline.png)

如上图所示，GLES的渲染大致分这六个步骤，其中将三角形坐标和颜色传递给GPU的是在 Vertex Shader 或者 Fragment Shader 中。而这两个 Shader 在高版本的 GLES中是支持可编程的，编程使用的语言就是上面说的 GLSL。
大概先说这么多，你暂时也不用深究其中的原理。目前你只要知道如何把数据给到GPU即可。

下面给出 Vertex Shader 和 Fragment Shader 的代码片段：
```java
    private static final String VERTEX_SHADER =
            "attribute vec4 aPosition;" +
                    "void main() {" +
                    "    gl_Position = aPosition;" +
                    "}";

    private static final String FRAGMENT_SHADER =
            "precision mediump float;" +
                    "uniform vec4 uColor;" +
                    "void main() {" +
                    "   gl_FragColor = uColor;" +
                    "}";
```
我们将这两个shader放入字符串中保存，其中 Vertex Shader 中的 aPostion 就是我们要存放三角形坐标的属性。而Fragment Shader 中的 uColor 则是存储三角形颜色的属性。

GLSL 的语言其实和 c 语言是很像的，要运行也是类似，需要经历编译链接。而经过编译链接后的产物则是 GLE S中的 Program 对象。下面看下代码：
```java
 public int createProgram(String vertexSource, String fragmentSource) {
        int vertexShader = loadShader(GLES32.GL_VERTEX_SHADER, vertexSource);
        if (vertexShader == 0) {
            return 0;
        }
        int pixelShader = loadShader(GLES32.GL_FRAGMENT_SHADER, fragmentSource);
        if (pixelShader == 0) {
            return 0;
        }

        int program = GLES32.glCreateProgram();
        GLES32.glAttachShader(program, vertexShader);
        GLES32.glAttachShader(program, pixelShader);
        GLES32.glLinkProgram(program);
        int[] linkStatus = new int[1];
        GLES32.glGetProgramiv(program, GLES32.GL_LINK_STATUS, linkStatus, 0);
        if (linkStatus[0] != GLES32.GL_TRUE) {
            GLES32.glDeleteProgram(program);
            program = 0;
        }
        return program;
    }


    public int loadShader(int shaderType, String source) {
        int shader = GLES32.glCreateShader(shaderType);
        GLES32.glShaderSource(shader, source);
        GLES32.glCompileShader(shader);
        int[] compiled = new int[1];
        GLES32.glGetShaderiv(shader, GLES32.GL_COMPILE_STATUS, compiled, 0);
        if (compiled[0] == 0) {
            GLES32.glDeleteShader(shader);
            GLES32.glGetShaderInfoLog(shader);
            shader = 0;
        }
        return shader;
    }
```
上面方法的作用是加载shader，编译shader，创建 Program，最后进行链接。

一旦Program被成功链接，那么我们就可以把三角形坐标和颜色赋值给程序里的 aPosition 和 uColor 了。具体怎么做呢？看如下代码：
```java
        GLES32.glUseProgram(program);                                                                                        //1
        int triangleColorLocation = GLES32.glGetUniformLocation(program, "uColor");                             //2
        GLES32.glUniform4fv(triangleColorLocation, 1, createFloatBuffer(TRIANGLE_COLOR));                     //3
        int aPositionLocation = GLES32.glGetAttribLocation(program, "aPosition");                                  //4
        GLES32.glEnableVertexAttribArray(aPositionLocation);                                                             //5
        FloatBuffer vertexBuffer = createFloatBuffer(TRIANGLE_COORDS);                                             //6
        GLES32.glVertexAttribPointer(aPositionLocation, 2, GLES32.GL_FLOAT, false, 2 * 4, vertexBuffer);      //7
```
这里一行一行解释：
1. 当 Program 链接成功后，如果要使用，需要调用 `glUseProgram`方法。
2. 我们可以看到 uColor 是被 uniform 修饰的，uniform 在 GLSL 中表示常量，我们可以过 `glGetUniformLocation`方法获取 uColor 在 GPU 常量区的位置。
3. 拿到 uColor 的位置后，使用 `glUniform4fv`即可对其进行赋值。
4. aPosition 被 `attribute`修饰，attribute 表示vertex shader 中的顶点属性，属于变量，用来存放顶点相关数据，这里我们用它存放三角形坐标。通过 `glGetAttribLocation` 方法，拿到了 aPostion 的位置。
5. 这里和 uniform 做法不同，首先我们要激活这个 attribute，然后才能给它赋值，激活方法是 `glEnableVertexAttribArray`，一旦激活后，就可以对他进行赋值了。
6. 这一步是对 aPosition 对应的顶点属性进行赋值，但由于这是一个坐标数组，我们需要确切告诉它我们坐标数组的大小和格式，只有这样GPU在运行时才能确切的拿到我们传入的三个坐标。这里使用了`glVertexAttribPointer`方法，其参数列表如下：
```
void glVertexAttribPointer(
    GLuint  	index,
 	GLint  	size,
 	GLenum  	type,
 	GLboolean  	normalized,
 	GLsizei  	stride,
 	const GLvoid *  	pointer);
```
上面这个是c语言的参数列表，其中 index 即 attribute 的 location，这里传 aPositionLocation。
size 表示每一个attribute 中每一个顶点的数据量，根据上面的坐标，每个顶点只有 x和y，即这里传 2。
type表示传入数据的类型，我们传的是float，所以这里传入 GLES32.GL_FLOAT 。
normallzed 表示要不要给我们传输的数据做归一化（转换到 -1~1之间），我们传入的已经是归一化后的数据，所以这里传 false 即可。
stride 表示 aPosition 这个顶点属性中，每两个顶点相隔的多少个字节，这里是两个 float，所以传入 4*2（float 占 4 个字节）。
pointer 表示 aPosition 要读取数据的地址或偏移。我们这里属于数据地址，所以传入 vertexBuffer 即可。属于偏移的在后面的章节中介绍。

调用完以上方法后，数据就被载入到 GPU 中了，接下来调用：
```java
        GLES32.glDrawArrays(GLES32.GL_TRIANGLE_STRIP, 0, 3);
        EGL14.eglSwapBuffers(mEGLDisplay, mEGLSurface);
```
即可将三角形绘制出来。效果如下：
![https://blog-1256162814.cos.ap-nanjing.myqcloud.com/opengl/gles_triangle.png](https://blog-1256162814.cos.ap-nanjing.myqcloud.com/opengl/gles_triangle.png)

本章节完整代码如下：
```java
package com.zql.mobile.graphics.opengl;

import android.opengl.EGL14;
import android.opengl.EGLConfig;
import android.opengl.EGLContext;
import android.opengl.EGLDisplay;
import android.opengl.EGLSurface;
import android.opengl.GLES32;
import android.os.Bundle;
import android.view.SurfaceHolder;
import android.view.SurfaceView;

import androidx.appcompat.app.AppCompatActivity;

import com.zql.mobile.graphics.R;

import java.nio.ByteBuffer;
import java.nio.ByteOrder;
import java.nio.FloatBuffer;

public class OpenGLES_VertexAttributePointer_Activity extends AppCompatActivity implements SurfaceHolder.Callback {

    private EGLDisplay mEGLDisplay;
    private EGLContext mEGLContext;
    private EGLSurface mEGLSurface;
    private EGLConfig mEGLConfig;
    private static final String VERTEX_SHADER =
            "attribute vec4 aPosition;" +
                    "void main() {" +
                    "    gl_Position = aPosition;" +
                    "}";

    private static final String FRAGMENT_SHADER =
            "precision mediump float;" +
                    "uniform vec4 uColor;" +
                    "void main() {" +
                    "   gl_FragColor = uColor;" +
                    "}";

    private static final float TRIANGLE_COORDS[] = {
            0.5f, 0.5f,   // 0 top
            -0.5f, 0.5f,   // 1 bottom left
            0f, -0.5f    // 2 bottom right
    };

    private static final float TRIANGLE_COLOR[] = {0.1F, 0.9F, 0.4F, 1.0F};

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_open_g_l_vertex_attribute_pointer);
        SurfaceView sv = findViewById(R.id.surface_view);
        sv.getHolder().addCallback(this);
    }

    private void initEGL(SurfaceHolder holder) {
        // get an EGL display connection
        mEGLDisplay = EGL14.eglGetDisplay(EGL14.EGL_DEFAULT_DISPLAY);

        int[] version = new int[2];
        // initialize EGL display connection
        EGL14.eglInitialize(mEGLDisplay, version, 0, version, 1);

        // target attribute list
        int[] attribList = {
                EGL14.EGL_RED_SIZE, 8,
                EGL14.EGL_GREEN_SIZE, 8,
                EGL14.EGL_BLUE_SIZE, 8,
                EGL14.EGL_ALPHA_SIZE, 8,
                EGL14.EGL_NONE
        };
        EGLConfig[] configs = new EGLConfig[1];
        int[] numConfigs = new int[1];
        //get appropriate EGL frame buffer configuration
        EGL14.eglChooseConfig(mEGLDisplay, attribList, 0, configs, 0, configs.length,
                numConfigs, 0);
        mEGLConfig = configs[0];

        int[] attrib3_list = {
                EGL14.EGL_CONTEXT_CLIENT_VERSION, 3,
                EGL14.EGL_NONE
        };

        //create an EGL rendering context
        mEGLContext = EGL14.eglCreateContext(mEGLDisplay, mEGLConfig, EGL14.EGL_NO_CONTEXT,
                attrib3_list, 0);

        // create an EGL window surface
        int[] surfaceAttribs = {
                EGL14.EGL_NONE
        };
        mEGLSurface = EGL14.eglCreateWindowSurface(mEGLDisplay, mEGLConfig, holder.getSurface(),
                surfaceAttribs, 0);

        // connect the context to the surface
        EGL14.eglMakeCurrent(mEGLDisplay, mEGLSurface, mEGLSurface, mEGLContext);
    }


    @Override
    public void surfaceCreated(SurfaceHolder holder) {
        initEGL(holder);
        GLES32.glClearColor(1.0F, 0F, 0F, 1F);
    }

    @Override
    public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {
        GLES32.glViewport(0, 0, width, height);
        GLES32.glClear(GLES32.GL_COLOR_BUFFER_BIT);
        int program = createProgram(VERTEX_SHADER, FRAGMENT_SHADER);
        GLES32.glUseProgram(program);
        int triangleColorLocation = GLES32.glGetUniformLocation(program, "uColor");
        GLES32.glUniform4fv(triangleColorLocation, 1, createFloatBuffer(TRIANGLE_COLOR));
        int aPositionLocation = GLES32.glGetAttribLocation(program, "aPosition");
        GLES32.glEnableVertexAttribArray(aPositionLocation);
        FloatBuffer vertexBuffer = createFloatBuffer(TRIANGLE_COORDS);
        GLES32.glVertexAttribPointer(aPositionLocation, 2, GLES32.GL_FLOAT, false, 2 * 4, vertexBuffer);
        GLES32.glDrawArrays(GLES32.GL_TRIANGLE_STRIP, 0, 3);
        EGL14.eglSwapBuffers(mEGLDisplay, mEGLSurface);
    }

    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {

    }

    public int createProgram(String vertexSource, String fragmentSource) {
        int vertexShader = loadShader(GLES32.GL_VERTEX_SHADER, vertexSource);
        if (vertexShader == 0) {
            return 0;
        }
        int pixelShader = loadShader(GLES32.GL_FRAGMENT_SHADER, fragmentSource);
        if (pixelShader == 0) {
            return 0;
        }

        int program = GLES32.glCreateProgram();
        GLES32.glAttachShader(program, vertexShader);
        GLES32.glAttachShader(program, pixelShader);
        GLES32.glLinkProgram(program);
        int[] linkStatus = new int[1];
        GLES32.glGetProgramiv(program, GLES32.GL_LINK_STATUS, linkStatus, 0);
        if (linkStatus[0] != GLES32.GL_TRUE) {
            GLES32.glDeleteProgram(program);
            program = 0;
        }
        return program;
    }


    public int loadShader(int shaderType, String source) {
        int shader = GLES32.glCreateShader(shaderType);
        GLES32.glShaderSource(shader, source);
        GLES32.glCompileShader(shader);
        int[] compiled = new int[1];
        GLES32.glGetShaderiv(shader, GLES32.GL_COMPILE_STATUS, compiled, 0);
        if (compiled[0] == 0) {
            GLES32.glDeleteShader(shader);
            GLES32.glGetShaderInfoLog(shader);
            shader = 0;
        }
        return shader;
    }

    public FloatBuffer createFloatBuffer(float[] coords) {
        ByteBuffer bb = ByteBuffer.allocateDirect(coords.length * 4);
        bb.order(ByteOrder.nativeOrder());
        FloatBuffer fb = bb.asFloatBuffer();
        fb.put(coords);
        fb.position(0);
        return fb;
    }
}

```

# 参考
[https://learnopengl-cn.github.io/01%20Getting%20started/04%20Hello%20Triangle/#_3](https://learnopengl-cn.github.io/01%20Getting%20started/04%20Hello%20Triangle/#_3)
[https://www.khronos.org/registry/OpenGL-Refpages/es3.0/html/glVertexAttribPointer.xhtml](https://www.khronos.org/registry/OpenGL-Refpages/es3.0/html/glVertexAttribPointer.xhtml)