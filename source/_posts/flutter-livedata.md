---
title: 在 Flutter 中简单实现 LiveData
date: 2019-9-11 16:09:25
tags: flutter
cover: https://blog-1256162814.cos.ap-nanjing.myqcloud.com/normal/flutter_livedata.png
---

# 在 Flutter 中简单实现 LiveData



如果你是一名 Android 开发者，[LiveData](https://developer.android.com/topic/libraries/architecture/livedata) 一定不陌生 ，它和 [ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel) 组合便可以构成简单的 MVVM 模式，而且是生命周期敏感的，不会造成不必要内存泄漏。

那么在 Flutter 中是否也能使用到 LiveData 呢？很可惜，官方并没有给出相关 API。但是使用现有的官方 API 和第三方 API 还是可以简单实现的。

首先我们总结下 LiveData 和 ViewModel 的几个特点：

1. LiveData 可自动通知其上的观察者其值的改变。
2. LiveData 会监听 Activity 或 Fragment 的生命周期，若视图已销毁，则不会通知其上的观察者（observeForeve除外）。
3. ViewModel 用于业务逻辑和 UI 解耦，可复用。

<!-- more -->

## LiveData 实现

```dart
import 'package:flutter/material.dart';

class LiveData<T> with ChangeNotifier {
  T _data;

  T get value => _data;

  set value(T data) {
    _data = data;
    notifyListeners();
  }
}
```

就是这么简单，一个 Flutter 版的 LiveData 已经实现了。

### LiveDataWidget 实现

仅仅使用 LiveData 还不够优雅，我们仍需手动调用 `setState` 方法去触发 Widget 刷新，接下来定义一个 LiveDataWidget 类，用于监听 LiveData 并自动更新其中的 Widget。

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

class LiveDataWidget<T extends LiveData> extends StatelessWidget {
  LiveDataWidget({Key key, @required this.data, @required this.builder})
      : super(key: key);

  final T data;

  final Widget Function(BuildContext context, T value, Widget child) builder;

  @override
  Widget build(BuildContext context) {
    return ChangeNotifierProvider<T>(
      builder: (_) {
        return data;
      },
      child: Consumer(builder: builder),
    );
  }
}
```

这里使用到了 `provider` 中的 `ChangeNotifierProvider` 和 `Consumer`，这样一个简单的组合就能做到当 `data`发生变化，Consumer 就会重新构建其中的 Widget。

## ViewModel 实现

`ViewModel` 并没有固定的实现，可以在实际使用中自行定义。



## Demo

```dart
import 'package:bili_livedata/bili_livedata.dart';
import 'package:flutter/material.dart';

void main() => runApp(MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: MyHomePage(title: 'Flutter Demo Home Page'),
    );
  }
}

///使用 mixin 来定义一个ViewModel
mixin HomePageViewModel {
  /// 用于保存计数的 LiveData
  final LiveData<int> counterLiveData = LiveData();

  /// 调用此方法，会在 [counterLiveData] 值的基础上加 [add] 这个值
  void counterAdd(int add) {
    if (counterLiveData.value == null) {
      counterLiveData.value = 0;
    } else {
      counterLiveData.value = counterLiveData.value + add;
    }
  }
}

// 使用 with 来并入 HomePageViewModel
class MyHomePage extends StatelessWidget with HomePageViewModel {
  MyHomePage({Key key, this.title}) : super(key: key);

  final String title;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(title),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            Text(
              'You have pushed the button this many times:',
            ),
            /// LiveDataWidget 的使用
            LiveDataWidget<LiveData<int>>(
                data: counterLiveData, // HomePageViewModel 中的 counterLiveData
                builder: (context, value, _) {
                  return Text(value.value.toString()); // 在 Text 上显示 counterLiveData 中的值。
                }),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          counterAdd(2); // 调用 HomePageViewModel 中的 counterAdd 方法
        },
        tooltip: 'Increment',
        child: Icon(Icons.add),
      ),
    );
  }
}

```

结合上面的代码再来回顾一下我们上面结合的三点：

1. LiveData 可自动通知其上的观察者其值的改变。

   **已实现** ，通过 ChangeNotifier 和 ChangeNotifierProvider 实现。

2. LiveData 会监听 Activity 或 Fragment 的生命周期，若视图已销毁，则不会通知其上的观察者（observeForeve除外）。

   **已实现**，ChangeNotifierProvider 内部会在 dispose 方法会去调用 ChangeNotifier 的 dispose 方法，所以当   页面销毁，ChangeNotifier（即 LiveData）上的监听者会被移除，不会造成内存泄漏。

3. ViewModel 用于业务逻辑和 UI 解耦，可复用。

   **已实现**，观察 HomePageViewModel，可以发现，业务逻辑都其中在其中，MyHomePage 只负责取 HomePageViewModel 中的 counterLiveData 值做展示，做到了业务和视图分离。另外 HomePageViewModel 作为一个 mixin，是可以被复用的，flutter 中也支持 with 多个 mixin。

以上，简单的 Flutter 版 LiveData 就已经实现。



## 总结

不管是否使用 LiveData，重要的是学习其中的设计思想：

1. 业务层视图层低耦合
2. 业务层高复用
3. UI 生命周期管理

另外将业务分离可以更有利于单元测试。