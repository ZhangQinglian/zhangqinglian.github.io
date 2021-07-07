---
title: Android官方架构组件介绍之ViewModel[翻译]
date: 2017-05-22 10:54:25
tags: android
cover: http://cdn.zqlxtt.cn/final-architecture.png
---

## ViewModel

像Activity，Fragment这类应用组件都有自己的生命周期并且是被Android的Framework所管理的。Framework可能会根据用户的一些操作和设备的状态对Activity或者Fragment进行销毁和重构。作为开发者，这些行为我们是无法干预的。

所以Activity或Fragment中的一些数据也会随着销毁而丢失，随着重构而重新生成。比如你的Activity中有个用户列表，当这个Activity重构的时候，新的Activity会重新获取用户列表。对于一些简单的数据，Activity可以使用`onSaveInstanceState()`方法，并从onCreate的bundle中重新获取。但这一方法途径仅仅适合一些简单的UI状态，对于用户列表这种庞大的数据并不适合。
<!-- more -->
还存在一个问题，Activity或者Fragment经常会做一些异步的耗时操作。随之就需要Activity和Fragment管理这些异步操作，并在自己被destroyed的时候清理它们，从而保证内存溢出这类问题的发生。这样的处理会随着项目扩大而变得十分复杂，一不留神，你的App就Crash了。

Activity和Fragment本身需要处理很多用户的输入事件并和操作系统打交道，所以当它们还要花时间管理它们的数据资源时，class文件就会变得异常庞大，然后就会造就出所谓的`god activities`和`god fragments`。这些UI控制类仅仅靠一个class就能处理相关的所有事务。简直跟上帝没啥两样。但这些类如果要进行单元测试的话，那就尴尬了。

所以就有了`MVC`,`MVP`这类设计模式，将视图与数据分离。今天讲到的`ViewModel`类的功能也一样，就是讲数据从UI中分离出来。并且当Activity或Fragment重构的时候，ViewModel会自动保留之前的数据并给新的Activity或Fragment使用。对于上面提到的用户列表的例子，ViewModel会为我们很好的管理这些数据。

<!--more-->

```java
public class MyViewModel extends ViewModel {
    private MutableLiveData<List<User>> users;
    public LiveData<List<User>> getUsers() {
        if (users == null) {
            users = new MutableLiveData<List<Users>>();
            loadUsers();
        }
        return users;
    }

    private void loadUsers() {
        // do async operation to fetch users
    }
}
```

接着在我们的Activity中就能这样使用了：

```java
public class MyActivity extends AppCompatActivity {
    public void onCreate(Bundle savedInstanceState) {
        MyViewModel model = ViewModelProviders.of(this).get(MyViewModel.class);
        model.getUsers().observe(this, users -> {
            // update UI
        });
    }
}
```

这是当MyActivity被重构时，获得到的model实例是与重构前同一个，当MyActivity被销毁时，Framework会调用ViewModel的`onCleared()`,我们就可以在此方法中做资源的清理。

>   因为ViewModel的生命周期是和Activity或Fragment分开的，所以在ViewModel中绝对不能引用任何View对象或者任何引用了Activity的Context的对象。如果ViewModel中需要Application的Context的话，可以继承`AndroidViewModel`。


## Fragment之间的数据共享

在Activity中包好多个Fragment并且需要相互通信是非常常见的，这时就需要这些Fragment定义一些接口，然后让Activity来进行协调。而且这些Fragment还需要处理其他Fragment不可见或者还没有创建这些细节问题。

上面这个动点可以被ViewModel轻易解决，想象意向有这么个Activity，它包含FragmentA和FragmentB，其中A是用户列表，B是用户的详细数据，点击列表上的某个用户，在B中显示相应的数据。

看看使用ViewModel怎么处理这个问题：

```java
public class SharedViewModel extends ViewModel {
    private final MutableLiveData<Item> selected = new MutableLiveData<Item>();

    public void select(Item item) {
        selected.setValue(item);
    }

    public LiveData<Item> getSelected() {
        return selected;
    }
}

public class MasterFragment extends Fragment {
    private SharedViewModel model;
    public void onActivityCreated() {
        model = ViewModelProviders.of(getActivity()).get(SharedViewModel.class);
        itemSelector.setOnClickListener(item -> {
            model.select(item);
        });
    }
}

public class DetailFragment extends LifecycleFragment {
    public void onActivityCreated() {
        SharedViewModel model = ViewModelProviders.of(getActivity()).get(SharedViewModel.class);
        model.getSelected().observe(this, { item ->
           // update UI
        });
    }
}
```

这里要注意的是两个Fragment都使用了`getActivity`作为参数来获得ViewModel实例。这表示这两个Fragment获得的ViewModel对象是同一个。

使用了ViewModel的好处如下：

- Activity不需要做任何事情，不需要干涉这两个Fragment之间的通信。
- Fragment不需要互相知道，即使一个消失不可见，另一个也能很好的工作。
- Fragment有自己的生命周期，它们之间互不干扰，即便你用一个FragmentC替代了B，FragmentA也能正常工作，没有任何问题。

## ViewModel的生命周期

ViewModel的生命周期跟着传递给`ViewModelProvider`的`LifeCycle`走，当生成了ViewModel的实例后，它会一直待在内存中，直到对应的LifeCycle彻底结束。下面是ViewModel与Activity对应的生命周期图：

![ViewModel生命周期](http://backup.flutter-dev.cn/viewmodel-lifecycle.png)
