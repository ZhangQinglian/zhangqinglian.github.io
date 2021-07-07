---
title: Android官方架构组件介绍之LifeCycle[翻译]
date: 2017-05-19 17:07:10
tags: android
cover: http://cdn.zqlxtt.cn/final-architecture.png
---

> Google 2017 I/O开发者大会于近日召开，在开发者大会上谷歌除了发布了Android O等一些新产品之外，也对Android代码的架构做出了一个官方的回应。

[Google 2017 I/O开发者大会Android架构组件介绍现场视频](http://www.bilibili.com/video/av10638148/index_8.html)
<!-- more -->
下面是官方提供的Android App开发的架构图：

![google官方Android架构图](http://cdn.zqlxtt.cn/final-architecture.png)

<!-- more -->
从上图可以看到一些关键字：`ViewModel`,`LiveData`,`Room`等。其实看了上面视频的会发现Google官方Android架构组件一共包括以下几个：

- LifeCycle ： 与Activity和Fragment的生命周期有关
- LiveData ：异步可订阅数据，也是生命周期感知
- ViewModel ：视图数据持有模型，也是生命周期感知
- Room ：SQLite抽象层，用于简化SQLite数据存储

这篇文章主要讲解LifeCycle在项目的简单用法。

## AS中添加依赖

首先在工程根目录的`build.gradle`中添加一下内容：

```groovy
allprojects {
    repositories {
        jcenter()
        maven { url 'https://maven.google.com' }  //添加此行
    }
}
```
然后在应用目录下的`build.gradle`中添加以下依赖：

```groovy
//For Lifecycles, LiveData, and ViewModel
compile "android.arch.lifecycle:runtime:1.0.0-alpha1"
compile "android.arch.lifecycle:extensions:1.0.0-alpha1"
annotationProcessor "android.arch.lifecycle:compiler:1.0.0-alpha1"

//For Room
compile "android.arch.persistence.room:runtime:1.0.0-alpha1"
annotationProcessor "android.arch.persistence.room:compiler:1.0.0-alpha1"
```

## LifeCycle相关使用

在我们平时的项目中经常会遇到很多需要依赖生命周期的逻辑处理，比如有这么一个需求。
在某个Activity我们需要在屏幕上展现用户的地理位置。简单的实现方法如下：

```java
class MyLocationListener {
    public MyLocationListener(Context context, Callback callback) {
        // ...
    }

    void start() {
        // connect to system location service
    }

    void stop() {
        // disconnect from system location service
    }
}

class MyActivity extends AppCompatActivity {
    private MyLocationListener myLocationListener;

    public void onCreate(...) {
        myLocationListener = new MyLocationListener(this, (location) -> {
            // update UI
        });
  }

    public void onStart() {
        super.onStart();
        myLocationListener.start();
    }

    public void onStop() {
        super.onStop();
        myLocationListener.stop();
    }
}
```

虽然以上代码看起来很简洁，但在实际项目中的onStart，onStop方法可能会变得相当庞大。
此外，实际情况可能并不像上面这么简单，例如我们需要在start位置监听前做用户状态检测，检测是一个耗时的任务，那么很有可能在检测结束前用户提前退出了Activity，这时候就会导致`myLocationListener.start()`在`myLocationListener.stop()`后面调用，从而引起很多难以定位的问题。代码如下：

```java
class MyActivity extends AppCompatActivity {
    private MyLocationListener myLocationListener;

    public void onCreate(...) {
        myLocationListener = new MyLocationListener(this, location -> {
            // update UI
        });
    }

    public void onStart() {
        super.onStart();
        Util.checkUserStatus(result -> {
            // what if this callback is invoked AFTER activity is stopped?
            if (result) {
                myLocationListener.start();
            }
        });
    }

    public void onStop() {
        super.onStop();
        myLocationListener.stop();
    }
}
```

这时候就该今天的主角`LifeCycle`出场了。它提供了一套接口帮助你处理这些问题。

### LifeCycle

`LifeCyle`类持有Activity或者Fragment的生命周期相关信息，并且支持其他对象监听这些状态。

`LifeCyle`有两个枚举用于追踪生命周期中的状态。

**Event**

这是生命周期的事件类，会在Framework和LifeCycle间传递，这些事件映射到Activity和Fragment的回调事件中。

**State**

LifeCycle所持有Activity或Fragment的当前状态。

一个类想要监听LifeCycle的状态，只需要给其方法加上注解：

```java
public class MyObserver implements LifecycleObserver {
    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    public void onResume() {
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    public void onPause() {
    }
}
aLifecycleOwner.getLifecycle().addObserver(new MyObserver());
```

### LifeCycleOwner

`LifeCycleOwner`是一个只有一个方法的接口用于表明其有一个LifeCycle对象。这个方法为`getLifecycle()`。
这个对象给Acitivity，Fragment和LifeCycle提供了一个很好的抽象关系，Activity和Fragment只要实现这个接口就能配合LifeCycle实现生命周期监听。

> 注意：由于目前LifeCycle处于alpha阶段，所以Fragment和AppCompatActivity并不会实现这些方法，在此之前，可以使用`LifecycleActivity`和`LifecycleFragment`。等LifeCycle趋于稳定后，Fragment和AppCompatActivity会默认实现这些。


对于之前的位置监听的例子，我们可以让`MyLocationListener`继承LifecycleObserver，在onCreate中使用LifeCycle进行初始化，剩下的问题则不必担心了。因为`MyLocationListener`有能力进行生命周期的判断。

```java
class MyActivity extends LifecycleActivity {
    private MyLocationListener myLocationListener;

    public void onCreate(...) {
        //此处进行初始化getLifecycle()传入LifeCycle对象
        myLocationListener = new MyLocationListener(this, getLifecycle(), location -> {
            // update UI
        });
        //检测用户状态并启用监听
        Util.checkUserStatus(result -> {
            if (result) {
                myLocationListener.enable();
            }
        });
  }
}

```
下面看一下MyLocationListener

```java
class MyLocationListener implements LifecycleObserver {
    private boolean enabled = false;
    public MyLocationListener(Context context, Lifecycle lifecycle, Callback callback) {
       ...
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    void start() {
        if (enabled) {
           // connect
        }
    }

    public void enable() {
        enabled = true;
        //⓵
        if (lifecycle.getState().isAtLeast(STARTED)) {
            // connect if not connected
        }
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    void stop() {
        // disconnect if connected
    }
}
```
LifeCycle最重要的特性就是在⓵处，他可以提供主动查询生命周期状态的方法。这样就避免了上面遇到的`myLocationListener.start()`在`myLocationListener.stop()`后面调用的问题。

通过以上的实现，我们的`LocationListener`是完全的生命周期感知了，它可以进行自己的初始化和资源清理而不必受Activity或者Fragment的管理。这时候如果我们在其他Activity或者Fragment中使用`LocationListener`，我们只需要初始化它就行了，不必再担心生命周期对它的影响，因为它内部会做好这一切。

通过LefeCycle工作的类我们称之为`生命周期感知`。鼓励需要使用Android生命周期的类的库提供生命周期感知组件，以便客户端可以轻松地在客户端上集成这些类，而无需手动生命周期管理。

`LiveData`就是生命周期感知组件的示例，将LiveData和ViewModel一起使用，可以在遵循Android生命周期的情况下，更容易的使用数据来填充UI。

## 生命周期的最佳实践

- 保持你的UI（Activity和Fragment）尽可能简洁。它们不应该试图获取它们的数据而是使用ViewModel来执行此操作，并通过LiveData的回调将数据更新到UI中。
- 尝试编写数据驱动的UI，你的UI的责任是在数据更改时更新视图，或将用户操作通知给ViewModel。
- 将你的数据逻辑放在ViewModel类中。 ViewModel应该作为UI和其他数据操作的连接器。值得注意的是，ViewModel并不负责提取数据（例如，从网络）。相反，ViewModel应该调用其他接口来执行此工作，然后将结果提供给UI。
- 使用`Data Binding`可以让你的的UI代码变得相当干净利落。这将使你的UI更具声明性，并最大限度地减少书写UI更新的代码。如果您更喜欢在Java中执行此操作，请使用像`Butter Knife`这样的库来避免使用样板代码并进行更好的抽象。
- 如果你有一个复杂的UI，请考虑创建一个Presenter类来处理UI修改。这通常是过度架构的，但可能有助于使你的UI更容易测试。
- 不要在ViewModel中引用View或Activity上下文。如果ViewModel在Activity或View销毁的情况下依旧存活，这时将导致内存泄漏。

## 补充
在自定义的Activity或Fragment中实现LifeCycleOwner，可以实现`LifecycleRegistryOwner`这个接口。而不是继承（LifeCycleFragment和LifeCycleActivity）

```java
public class MyFragment extends Fragment implements LifecycleRegistryOwner {
    LifecycleRegistry lifecycleRegistry = new LifecycleRegistry(this);

    @Override
    public LifecycleRegistry getLifecycle() {
        return lifecycleRegistry;
    }
}
```

如果你要在自定义的类中实现LifeCycleOwner，可以使用`LifecycleRegistry`,但是你需要主动向其转发生命周期的事件。但如果你自定义类是Fragment和Activity的话并且它们实现的是LifecycleRegistryOwner，那么事件转发都是自动完成的。
