---
title: Android官方架构组件介绍之LiveData[翻译]
date: 2017-05-21 15:12:01
tags: android
cover: http://cdn.zqlxtt.cn/final-architecture.png
---

## LiveData

`LiveData`是一个用于持有数据并支持数据可被监听（观察）。和传统的观察者模式中的被观察者不一样，LiveData是一个`生命周期感知`组件，因此观察者可以指定某一个`LifeCycle`给LiveData，并对数据进行监听。

如果观察者指定`LifeCycle`处于`Started`或者`RESUMED`状态，LiveData会将观察者视为活动状态，并通知其数据的变化。
<!-- more -->
我们看一段代码：

```java
public class LocationLiveData extends LiveData<Location> {
    private LocationManager locationManager;

    private SimpleLocationListener listener = new SimpleLocationListener() {
        @Override
        public void onLocationChanged(Location location) {
            setValue(location);
        }
    };

    public LocationLiveData(Context context) {
        locationManager = (LocationManager) context.getSystemService(
                Context.LOCATION_SERVICE);
    }

    @Override
    protected void onActive() {
        locationManager.requestLocationUpdates(LocationManager.GPS_PROVIDER, 0, 0, listener);
    }

    @Override
    protected void onInactive() {
        locationManager.removeUpdates(listener);
    }
}
```

上面有三个值得注意的地方：

<!--  more -->
- onActive()

当这个方法被调用时，表示LiveData的观察者数量从0变为了1，这时就我们的位置监听来说，就应该注册我们的时间监听了。

- onInactive()

这个方法被调用时，表示LiveData的观察者数量变为了0，既然没有了观察者，也就没有理由再做监听，此时我们就应该将位置监听移除。

- setValue()

通过调用这个方法来更新LiveData的数据，并通知处于活动状态的观察者。

接着我们就能像下面这样使用LocationLiveData了。

```java
public class MyFragment extends LifecycleFragment {
    public void onActivityCreated (Bundle savedInstanceState) {
        LiveData<Location> myLocationListener = ...;
        Util.checkUserStatus(result -> {
            if (result) {
                myLocationListener.addObserver(this, location -> {
                    // update UI
                });
            }
        });
    }
}
```

注意上面的`addObserver`方法，我们将`LifeCycleOwner`作为第一个参数传递了进去，这表示我们的LocationLiveData将遵照这个Fragment所持有的LifeCycle办事。

- 如果LifeCycle不在Started或者RESUMED这两个状态，那么观察者将无法接受到数据更新的回调，即使数据发生了变化。
- 如果LifeCycle销毁了，即生命周期结束，观察者将被自动从LiveData中移除。

既然LocationLiveData是生命周期感知的，那么我们就可以稍微改动一下它的代码，让它可以被多个Activity或者Fragment公用：

```java
public class LocationLiveData extends LiveData<Location> {
    private static LocationLiveData sInstance;
    private LocationManager locationManager;

    @MainThread
    public static LocationLiveData get(Context context) {
        if (sInstance == null) {
            sInstance = new LocationLiveData(context.getApplicationContext());
        }
        return sInstance;
    }

    private SimpleLocationListener listener = new SimpleLocationListener() {
        @Override
        public void onLocationChanged(Location location) {
            setValue(location);
        }
    };

    private LocationLiveData(Context context) {
        locationManager = (LocationManager) context.getSystemService(
                Context.LOCATION_SERVICE);
    }

    @Override
    protected void onActive() {
        locationManager.requestLocationUpdates(LocationManager.GPS_PROVIDER, 0, 0, listener);
    }

    @Override
    protected void onInactive() {
        locationManager.removeUpdates(listener);
    }
}
```
这里使用单例的原因就是让多个Activity或者Fragment共享一个LocationLiveData实例。
然后我们可以这么使用：

```java
public class MyFragment extends LifecycleFragment {
    public void onActivityCreated (Bundle savedInstanceState) {
        Util.checkUserStatus(result -> {
            if (result) {
                MyLocationListener.get(getActivity()).addObserver(this, location -> {
                   // update UI
                });
            }
        });
  }
}
```

通过这么一改，现在即使有多个Activity或者Fragment在使用LocationLiveData，它也能对其进行优雅的管理。不必理会页面销毁带来的诸多麻烦。

总结几点LiveData的有点：

- 没有内存溢出

当观察者被绑定他们对应的LifeCycle以后，当页面销毁时他们会自动被溢出，不会导致内存溢出。

- 不会因为Activity的不可见导致Crash

当Activity不可见时，即使有数据变化，LiveData也不会通知观察者。因为此时观察者的LifeCyele并不处于Started或者RESUMED状态。

- 配置的改变

当当前Activity配置改变（如屏幕方向），导致重新从onCreate走一遍，这是观察者们会立刻收到配置变化前的最新数据。

- 资源共享

我们只需要一个LocationLivaData,连接系统服务一次，就能支持所有的观察者。

- 不再有人为生命周期处理

通过上面的代码可以知道，我们的Activity或者Fragment只要在需要观察数据的时候观察数据即可，不需要理会生命周期变化了。这一切都交给LiveData来自动管理。

## LiveData的转换

有时候有这样的需求，需要在LiveData将变化的数据通知给观察者前，改变数据的类型；或者是返回一个不一样的LiveData。

这里介绍一个类`Transformations`,它可以帮助完成上面的这些操作。

- Transformations.map()

在LiveData数据的改变传递到观察者之前，在数据上应用一个方法：

```java
LiveData<User> userLiveData = ...;
LiveData<String> userName = Transformations.map(userLiveData, user -> {
    user.name + " " + user.lastName
});
```

这里我们如果只需要知道变化用户的名字，那么只要观察userName这个LiveData对象即可。它会从userLiveData数据中提取用户名并传递给它自己的观察者。

- Transformations.switchMap()

与Transformations.map()类似，只不过这里传递个switchMap()的方法必须返回一个LiveData对象。

```java
private LiveData<User> getUser(String id) {
  ...;
}

LiveData<String> userId = ...;
LiveData<User> user = Transformations.switchMap(userId, id -> getUser(id) );
```

当你考虑在ViewModel中使用LifeCycle对象时，这种转换就是一个可选的解决方案。
假如有一下需求，用户输入一个地址，我们在屏幕上更新这个地址对应的邮编，简单的写法如下：

```java
class MyViewModel extends ViewModel {
    private final PostalCodeRepository repository;
    public MyViewModel(PostalCodeRepository repository) {
       this.repository = repository;
    }

    private LiveData<String> getPostalCode(String address) {
       // DON'T DO THIS
       return repository.getPostCode(address);
    }
}
```
这样写问题显然很严重，当每次调用getPostalCode方法后，UI代码中都需要对getPostalCode的返回值做注册观察者操作，并且还要移除上一个观察者，这样显然是低效率的。此外，如果这时UI因为配置的变化（屏幕旋转）重建了，那么它会触发再次调用getPostalCode，而不是使用之前的调用结果。

因此我们可以做如下转换：

```java
class MyViewModel extends ViewModel {
    private final PostalCodeRepository repository;
    private final MutableLiveData<String> addressInput = new MutableLiveData();
    public final LiveData<String> postalCode =
            Transformations.switchMap(addressInput, (address) -> {
                return repository.getPostCode(address);
             });

  public MyViewModel(PostalCodeRepository repository) {
      this.repository = repository
  }

  private void setInput(String address) {
      addressInput.setValue(address);
  }
}
```

注意，这里我们将postalCode访问限制符写成public final，因为它将始终不变，UI只要在需要用的时候将观察者注册到postalCode中就行。这是当用户调用setInput后，如果postalCode上有可活动的观察者，那么repository.getPostCode(address)就会被调用，如果此时没有可活动的观察者，则repository.getPostCode(address)不会被调用。

