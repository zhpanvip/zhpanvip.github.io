---
layout: article
index_img: https://raw.githubusercontent.com/zhpanvip/images/master/blog/img/android_arch.png
title: Android 架构思想与 MVVM 框架封装
date: 2022-08-20 14:23:28
categories:
- Android进阶
tags: [MVC,MVP,MVVM]
---


关于Android项目架构也是一个老生常谈的话题了，网上关于Android架构的文章不胜枚举，但是通过Google检索关键字，首页的热门文章多数是对于MVC、MVP及MVVM等架构的概念介绍，概念性的文章对于不了解Android架构的同学来说并不一定能起到很好的帮助。本篇文章其实源自笔者在公司内部的技术分享，稍作修改后作为文章发布出来。文章内容涉及从
MVC、MVP 到 MVVM 的演化，同时为便于理解，每种架构都做了代码演示，最后基于 Jetpack 提供的组件封装了 MVVM
架构。文章内容比较基础，几乎没有晦涩难懂的知识，对于想要了解Android架构的同学会有很大的帮助。

## 一、Android 项目架构的演化

首先，我们应该明白一点，对于架构而言并不分平台。不管MVC、MVP 还是 MVVM
都不是Android平台独有的，甚至由于Android平台起步较晚，Android项目的架构或多或少的参考了前端的架构实现。

对于前端或者Android端项目而言代码可以分为三部分，分别为UI部分、业务逻辑部分以及数据控制部分。这三部分流转的起点来自于用户输入，即用户通过操作UI调起对应的业务逻辑获取数据，并最终将数据反馈到UI界面上，其流转图如下图所示。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/49e8e5c99ecb41ea9ac3169f940b8f67~tplv-k3u1fbpfcp-watermark.image?)

根据这三部分内容，我们可以将代码分为三层，即最初的MVC架构分层。但是随着项目的业务逐渐复杂，MVC架构的弊端显露，不能够支撑已有的业务。于是在此背景下衍生出了MVP架构来弥补MVC的不足。甚至后来谷歌官方推出的部分Jetpack组件来专门解决Android架构问题，从主推MVVM，到如今主推的MVI架构。但是不管架构如何演变，都脱离不了上述提到的三层逻辑，只不过新的架构在已有的基础上弥补了老架构的不足，让项目代码更容易维护。

接下来的内容我们主要探讨Android项目架构从MVC到MVVM的演化。

1.  ### MVC 简介

基于上面提到的三层逻辑，最初的Android项目采用的是MVC架构。MVC是 **Model-View-Controller**
的简称。简单来说MVC是用户操作View，View调用Controller去操作Model层，然后Model层将数据返回给View层展示。

- 模型层(Model) 负责与数据库和网络层通信，并获取和存储应用的数据；
- 视图层(View) 负责将 Model 层的数据做可视化的处理，同时处理与用户的交互；
- 控制层(Controller) 用于建立Model层与View层的联系，负责处理主要的业务逻辑，并根据用户操作更新 Model 层数据。

MVC 的结构图如下图所示。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7ea11f97de5a43bebf35d8aa1573335b~tplv-k3u1fbpfcp-watermark.image?)

在 Android 项目的 MVC 架构中由于 Activity 同时充当了 View 层与 Controller 层两个角色，所以 Android 中的 MVC 更像下面的这种结构：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/562c78eaf71942498b2865bdd34935e5~tplv-k3u1fbpfcp-watermark.image?)

基于上图，我们可以以APP的登录流程为例实现 Android 中 MVC 架构的代码。

#### （1）Model 层代码实现

Model 层用于处理登录请求并从服务器获取登录相关数据，成功后通知 View 层更新界面。代码如下：

```java
public class LoginModel {
    public void login(String username, String password, LoginListener listener) {
        RetrofitManager.getApiService()
                .login(username, password)
                .enqueue(new Callback<User>() {
                    @Override
                    public void onResponse(Call<User> call, Response<User> response) {
                        listener.onSuccess(response.data);
                    }

                    @Override
                    public void onFailed(Exception e) {
                        listener.onFailed(e.getMessage());
                    }
                });
        
    }
}
```

上述代码通过 LoginListener 来通知 View 层更新UI，LoginListener 是一个接口，如下：

```java
    public interface LoginListener {
    void onSuccess(User data);

    void onFailed(String msg);
}
```

#### （2）Controller/View 代码实现

由于 Android 的 MVC 架构中 Controller 与 View 层都是由 Activity 负责的，因此 Activity 需要实现 LoginListener 用来更新
UI。其代码如下：

```java
public class MainActivity extends AppCompactActivity implements LoginListener {

    private LoginModel loginModel;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        model = new LoginModel();
        findViewbyId(R.id.btn_fetch_data).setOnClickListener(view -> {
                    String username = findViewById(R.id.et_username).getText().toString();
                    String password = findViewById(R.id.et_password).getText().toString();
                    loginModel.login(username, password, this);
                }
        );
    }

    @Override
    public void onSuccess(User data) {
        // Update UI
    }

    @Override
    public void onFailed(String msg) {
        // Update UI
    }
}
```

从上述代码中可以看到，Android中的MVC代码对于分层并不明确，导致了Controller层与View层混为一体。与此同时，大家在写Android代码的时候一般不会刻意的再去抽出一个Model层，而是将
Model 层的代码也一股脑的塞到 Activity 中实现，

```java
public class MainActivity extends AppCompactActivity implements LoginListener, View.OnClickListener {

    private LoginModel loginModel;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        model = new LoginModel();

        findViewbyId(R.id.btn_fetch_data).setOnClickListener(view -> {
                    showLoading();
                    String username = findViewById(R.id.et_username).getText().toString();
                    String password = findViewById(R.id.et_password).getText().toString();
                    RetrofitManager.getApiService()
                            .login(username, password)
                            .enqueue(new Callback<User>() {
                                @Override
                                public void onResponse(Call<User> call, Response<User> response) {
                                    onSuccess(response.data);
                                }

                                @Override
                                public void onFailed(Exception e) {
                                    listener.onFailed(e.getMessage());
                                }
                            });
                }
        );
    }

    @Override
    public void onSuccess(User data) {
        dismissLoading();
        // Update UI
    }

    @Override
    public void onFailed(String msg) {
        dismissLoading();
        // Update UI
    }

    public void showLoading() {
        // ...
    }

    public void dismissLoading() {
        // ...
    }
}
```

这样的代码分层变得更加模糊，同时也使得代码的结构层次更加混乱。致使一些业务比较复杂的页面 Activity 中的代码可能多达数千行。由此可以看出 Android 项目中的 MVC
架构是存在很多问题的。总结主要有如下几点：

- Activity/Fragment 同时承担了 View 层与 Controller 层的工作，违背了单一职责；
- Model 层与 View 层存在耦合，存在相互依赖关系；
- 开发时不注重分层，Model层代码也被塞进了Activity/Fragment，使得代码层次更加混乱。


2.  ### MVP 简介

针对以上MVC架构中存在的问题，我们可以在MVC的基础上进行优化解决。即从Activity中剥离出控制层的逻辑，并阻断Model层与View层的耦合，Model层不直接与View通信，而是在数据改变时让
Model通知控制控制层，控制层再通知View层去做界面更新，这就是MVP的架构思想。MVP 是 **Model-View-Presenter** 的简称。 简单来说 MVP 就是将 MVC 的
Controller 改为 Presenter，即把逻辑层的代码从 Activity 中抽离到了 Presenter 中，这样代码层次变得更加清晰，其次 Model 层不再持有 View
层，代码更加解耦。

- 模型层(Model) 与MVC中的一致，同样是负责与数据库和网络层通信，并获取和存储应用的数据，区别在于Model层不再与View层直接通信，而是与Presenter层通信。
- 视图层(View) 负责将 Model 层的数据做可视化的处理，同时与Presenter层交互。跟MVC相比，MVP的View层与Model层不再耦合。
- 控制层(Presenter) 主要负责处理业务逻辑，并监听Model层数据改变，然后调用View层刷新UI。

MVP 的结构图如下图所示。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/47edd5445b484e8dbcbb0d63ad83a7fe~tplv-k3u1fbpfcp-watermark.image?)

从上图中可以看出，View直接与Presenter层通信，当View层接收到用户操作后会调用
Presenter层去处理业务逻辑。接着Presenter层通过Model去获取数据，Model层获取到数据后又将最新的数据传回 Presenter。
由于Presenter层又持有View层的引用，进而将数据传给`View`层进行展示。

下面我们我们仍然以登录为例通过代码来演示MVP架构的实现。

#### （1）Model 层代码实现

MVP中的Model层与MVC中的Model层是一致的，代码如下:

```java
public class LoginModel {
    public void login(String username, String password, LoginListener listener) {
        RetrofitManager.getApiService()
                .login(username, password)
                .enqueue(new Callback<User>() {
                    @Override
                    public void onResponse(Call<User> call, Response<User> response) {
                        listener.onSuccess(response.data);
                    }

                    @Override
                    public void onFailed(Exception e) {
                        listener.onFailed(e.getMessage());
                    }
                });
    }
}
```

在登录接口返回结果后通过 LoginListener 将结果回调出来。

```java
public interface LoginListener {
    void onSuccess(User user);

    void onFailed(String errorInfo);
}
```

#### （2）Presenter 层代码实现

由于 Presenter 需要通知 View 层更新UI，因此需要持有View，这里可以抽象出一个 View 接口。如下：

```java
public interface ILoginView {
    void showLoading();

    void dismissLoading();

    void loginSuccess(User data);

    void loginFailer(String errorMsg);
}
```

另外，Presenter也需要与Model层通信，因此Presenter层也会持有Model层，在用户触发登录操作后，调用Presenter的登录逻辑，Presenter通过Model进行登录操作，登录成功后再将结果反馈给View层更新界面。代码实现如下：

```java
public class LoginPresenter {

    // Model 层 
    private LoginModel model;

    // View 层 
    private ILoginView view;

    public LoginPresenter(LoginView view) {
        this.model = new LoginModel();
        this.view = view;
    }

    public void login(String username, String password) {
        view.showLoading();
        model.login(username, password, new LoginListener() {
            @Override
            public void onSuccess(User user) {
                view.loginSuccess(user);
                view.dismissLoading();
            }

            @Override
            public void onFailed(String msg) {
                view.loginFailer(msg);
                view.dismissLoading();
            }
        });
    }

    @Override
    public void onDestory() {
        // recycle instance.
    }
}

```

#### （3）View 层代码实现

Activity作为View层需要实现上述ILoginView接口，且View层需要持有Presenter来处理业务逻辑。View层的代码实现就变得非常简单了。

```java
public class MainActivity extends AppCompatActivity implements ILoginView {

    private LoginPresenter presenter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        presenter = new LoginPresenter(this);
        findViewById(R.id.button).setOnClickListener(v -> {
            String username = findViewById(R.id.et_username).getText().toString();
            String password = findViewById(R.id.et_password).getText().toString();
            presenter.login(username, password);
        });
    }

    @Override
    public void loginSuccess(User user) {
        // Update UI.
    }

    @Override
    public void loginFailer(String msg) {
        // Update UI.
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        presenter.onDestroy();
    }

    @Override
    public void showLoading() {

    }

    @Override
    public void dismissLoading() {
    }
}
```

MVP的本质是面向接口编程，实现依赖倒置原则。可以看得出来MVP架构在分层上相比 MVC更加清晰明确，且解耦了Model层与View层。但MVP也存在一些弊端，列举如下：

- 引入大量的 IView、IPresenter接口实现类，增加项目的复杂度。
- View 层与 Presenter 相互持有，存在耦合。

3.  ### MVVM 简介

MVP相比于MVC无疑有很多优点，但其自身也并非无懈可击，再加之MVP也并非Google官方推荐的架构，因此也只能算得上程序员对于Android项目架构优化的野路子。从2015年开始，Google官方开始对Android项目架构做出指导，随后推出DataBinding组件以及后边一系列的Jetpack组件来帮助开发者优化项目架构。Google官方给出的这一套指导架构就是MVVM。MVVM是 **
Model-View-ViewModel** 的简称。这一架构在一定程度上解决了MVP架构中存在的问题。虽然近期官方的指导架构由MVVM变为了MVI，但MVVM依然是目前Android项目的主流架构。

- 模型层(Model) 与MVP中的Model层一致，负责与数据库和网络层通信，获取并存储数据。与MVP的区别在于Model层不再通过回调通知业务逻辑层数据改变，而是通过观察者模式实现。
- 视图(View) 负责将Model层的数据做可视化的处理，同时与ViewModel层交互。
- 视图模型(ViewModel) 主要负责业务逻辑的处理，同时与 Model 层 和 View层交互。与MVP的Presenter相比，ViewModel不再依赖View，使得解耦更加彻底。

MVVM 架构的结构图如下。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dee5d989ca3a421eaa5386bc83b01dcd~tplv-k3u1fbpfcp-watermark.image?)

MVVM架构的本质是数据驱动，它的最大的特点是单向依赖。MVVM架构通过观察者模式让ViewModel与View解耦，实现了View依赖ViewModel，ViewModel依赖Model的单向依赖。

接下来我们仍然以登录为例，通过观察者模式来实现MVVM的架构代码。

#### （1）观察者模式

由于 MVVM 架构需要将 ViewModel 与 View 解耦。因此，这里可以使用观察者模式来实现。下面实现观察者模式的代码。

- 实现观察者接口

```java
public interface Observer<T> {
    // 成功的回调
    void onSuccess(T data);

    // 失败的回调
    void onFailed(String msg);
}
```

- 抽象被观察者

```java
public abstract class Observable<T> {
    // 注册观察者
    void register(Observer observer);

    // 取消注册
    void unregister(Observer observer);

    // 数据改变（设置成功数据）
    protected void setValue(T data) {

    }

    // 失败，设置失败原因
    void onFailed(String msg);
}
```

这里我们需要注意被观察者Observable中的setValue方法被设置成了protect。意味着如果View层拿到的是Observable实例，则无法调用setValue来设置数据，以此来保证代码的安全性。

- 被观察者具体实现

```java
public class LoginObservable implements Observable<User> {

    private final List<Observer<User>> list = new ArrayList<>();

    private User user;

    @Override
    public void register(Observer<User> observer) {
        list.add(observer);
    }

    @Override
    public void unregister(Observer<User> observer) {
        liset.remove(observer);
    }

    @Override
    public void setValue(User user) {
        this.user = user;
        for (Observer observer : list) {
            observer.onSuccess(user);
        }
    }

    @Override
    public void onFailed(String msg) {
        for (Observer observer : list) {
            observer.onFailed(msg);
        }
    }
}
```

在被观察者的实现类中setValue方法被设置为了public，意味着如果是LoginObservable，那么可以通过setValue来对其设置数据。

#### （2）Model 层代码实现

Model层代码与MVP/MVC的实现基本一致，如下：

```java
public class LoginModel {
    public void login(String username, String password, LoginObservable observable) {
        RetrofitManager.getApiService()
                .login(username, password)
                .enqueue(new Callback<User>() {
                    @Override
                    public void onResponse(Call<User> call, Response<User> response) {
                        observable.setValue(response.data);
                    }

                    @Override
                    public void onFailed(Exception e) {
                        observable.onFailed(e.getMessage());
                    }
                });
    }
}
```

需要注意的是LoginModel的login方法中接收的是LoginObservable类型，因此这里可以通过setValue来设置数据。

#### （3）ViewModel 层实现

ViewModel层需要持有Model层，并且ViewModel层持有一个LoginObservable，并开放一个getObservable的方法供View层使用。代码如下：

```java
public class LoginViewModel {

    private LoginObservable observable;

    private LoginModel loginModel;

    public LoginViewModel() {
        loginModel = new LoginModel();
        observable = new LoginObservable();
    }

    public void login(String username, String password) {
        loginModel.login(username, password, observable);
    }

    // 这里返回值是父类Observable，即View层得到的Observable无法调用setValue
    public Observable getObservable() {
        return observable;
    }
}
```

这里需要注意的是getObservable方法的返回值是Observable，意味着View层只能调用register方法来观察数据改变，而无法通过setValue来设置数据。

#### （4）View 层代码实现

View层持有ViewModel，用户触发登录事件通过View层交给Model处理，Model层在登录成后将数据交给ViewModel中的观察者。因此，View层只需要获取到ViewModel层的观察者并观察到数据改变后更新UI即可。代码如下：

```java
public class MainActivity extends AppCompatActivity implements Observer<User> {

    private LoginViewModel viewModel;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        viewModel = new LoginViewModel();
        viewModel.getObservable().register(this);
        findViewById(R.id.button).setOnClickListener(v -> {
            String username = findViewById(R.id.et_username).getText().toString();
            String password = findViewById(R.id.et_password).getText().toString();
            viewModel.login(username, password);
        });
    }

    @Override
    public void onDestroy() {
        super.onDestory();
        viewModel.getObservable().unregister(this);
    }

    @Override
    public void onSuccess(User user) {
        // fetch data success,update UI.
    }

    @Override
    public void onFailed(String message) {
        // fetch data failed,update UI.
    }
}
```

可以看到我们通过观察者模式实现了View->ViewModel->
Model的单向依赖。相比于MVP，MVVM解耦的更加纯粹。但是，上边的观察者模式是我们自己实现的，如果直接用到项目中肯定是不稳妥的。上面我们提到Google已经为我们提供了一套Jetpack组件来帮助开发者实现MVVM架构。那接下来我们就来认识一下通过Jetpack实现的MVVM架构。

## 二、使用 Jetpack 组件封装 MVVM 架构

如果你已经看到这里了，相信你对项目架构已经有了一定的认识。这里需要再强调一点架构是一种思想，它与我们使用什么工具来实现没有关系。就像上一章中我们通过自己写的观察者模式来实现MVVM一样，只要遵循了这个架构思想，那它就属于这一架构。Google官方推出的这些组件可以更高效的帮助我们来实现MVVM。

### 1. Jetpack MVVM 简介

MVVM 是 Google 官方推荐的框架。为此，Google 提供了一系列实现 MVVM 的工具，这些工具都被包含在Jetpack 组件中，例如 LiveData、ViewModel以及
DataBinding 等。下面是官方给的一个通过Jetpack组件实现的 MVVM 架构图。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f9efcc24a9564d6da173784dd6880703~tplv-k3u1fbpfcp-zoom-1.image)

这张图与我们上一章的MVVM结构图是一致的，只不过这里融入了Jetpack组件。可以看到图中的箭头都是单向的，View层指向了ViewModel层，表示View层持有ViewModel层的引用，但ViewModel层不持有View层。ViewModel层持有Repository层，但Repository层不会持有ViewModel。这张图与MVVM的对应关系如下：

- 视图(View) 对应图中的Activity/Fragment，包含了布局文件等于界面相关的东西；
- 视图模型(ViewModel) 对应图中的Jetpack ViewModel
  与LiveData，用于持有与UI相关的数据，且能保证在旋转屏幕后不丢失数据。另外还提供了给View层调用的接口以及和Repository通信的接口；
- 模型层(Model) 对应图中的 Repository，包含本地数据与服务端数据。

### 2. MVVM 框架代码封装

在了解了Jetpack MVVM后，为了更加高效的开发我们通常会做一些基础封装。例如如何结合网络请求与数据库来获取数据、Loading的显示逻辑等。本章内容要求读者对Jetpack
ViewModel、LiveData等组件有一定的了解。

#### 2.1 网络层封装

针对服务器返回的数据进行封装，可以抽出来一个带有泛型的Response基类。如下：

```kotlin
class BaseResponse<T>(
    var errorCode: Int = -1,
    var errorMsg: String? = null,
    var data: T? = null,
    var dataState: DataState? = null,
    var exception: Throwable? = null,
) {
    companion object {
        const val ERROR_CODE_SUCCESS = 0
    }
    
    val success: Boolean
        get() = errorCode == ERROR_CODE_SUCCESS
}

/**
 * 网络请求状态
 */
enum class DataState {
    STATE_LOADING, // 开始请求
    STATE_SUCCESS, // 服务器请求成功
    STATE_EMPTY, // 服务器返回数据为null
    STATE_FAILED, // 接口请求成功但是服务器返回error
    STATE_ERROR, // 请求失败
    STATE_FINISH, // 请求结束
}
```

对应网络请求的异常情况一般我们都会进行统一处理。这里我们可以放在Observer中进行。 对LiveData 的 Observer进行封装，从而实现Response与异常的统一处理。

```kotlin
abstract class ResponseObserver<T> : Observer<BaseResponse<T>> {
    
    final override fun onChanged(response: BaseResponse<T>?) {
        response?.let {
            when (response.dataState) {
                DataState.STATE_SUCCESS, DataState.STATE_EMPTY -> {
                    onSuccess(response.data)
                }
                DataState.STATE_FAILED -> {
                    onFailure(response.errorMsg, response.errorCode)
                }
                DataState.STATE_ERROR -> {
                    onException(response.exception)
                }
                else -> {
                }
            }
        }
    }
    
    private fun onException(exception: Throwable?) {
        ToastUtils.showToast(exception.toString())
    }
    
    /**
     * 请求成功
     *  @param  data 请求数据
     */
    abstract fun onSuccess(data: T?)
    
    /**
     * 请求失败
     *  @param  errorCode 错误码
     *  @param  errorMsg 错误信息
     */
    open fun onFailure(errorMsg: String?, errorCode: Int) {
        ToastUtils.showToast("Login Failed,errorCode:$errorCode,errorMsg:$errorMsg")
    }
}
```

封装 RetrofitCreator 用来创建 API Service。

```kotlin
object RetrofitCreator {
    
    private val mOkClient = OkHttpClient.Builder()
        .callTimeout(Config.DEFAULT_TIMEOUT, TimeUnit.MILLISECONDS)
        .connectTimeout(Config.DEFAULT_TIMEOUT, TimeUnit.MILLISECONDS)
        .readTimeout(Config.DEFAULT_TIMEOUT, TimeUnit.MILLISECONDS)
        .writeTimeout(Config.DEFAULT_TIMEOUT, TimeUnit.MILLISECONDS)
        .retryOnConnectionFailure(true)
        .followRedirects(false)
        .addInterceptor(HttpHeaderInterceptor())
        .addInterceptor(LogInterceptor())
        .build()
    
    private fun getRetrofitBuilder(baseUrl: String): Retrofit.Builder {
        return Retrofit.Builder()
            .baseUrl(baseUrl)
            .client(mOkClient)
            .addConverterFactory(GsonConverterFactory.create())
    }
    
    /**
     * Create Api service
     *  @param  cls Api Service
     *  @param  baseUrl Base Url
     */
    fun <T> getApiService(cls: Class<T>, baseUrl: String): T {
        val retrofit = getRetrofitBuilder(
            baseUrl
        ).build()
        return retrofit.create(cls)
    }
}
```

#### 2.2 Model 层封装

官方代码中的 Model 层是通过Repository实现的，为了减少模板代码，我们可以封装 BaseRepository 来处理网络请求。

```kotlin
open class BaseRepository {
    
    // Loading 状态的 LiveData
    val loadingStateLiveData: MutableLiveData<LoadingState> by lazy {
        MutableLiveData<LoadingState>()
    }
    
    /**
     * 发起请求
     *  @param  block 真正执行的函数回调
     *  @param responseLiveData 观察请求结果的LiveData
     */
    suspend fun <T : Any> executeRequest(
        block: suspend () -> BaseResponse<T>,
        responseLiveData: ResponseMutableLiveData<T>,
        showLoading: Boolean = true,
        loadingMsg: String? = null,
    ) {
        var response = BaseResponse<T>()
        try {
            if (showLoading) {
                loadingStateLiveData.postValue(LoadingState(loadingMsg, DataState.STATE_LOADING))
            }
            response = block.invoke()
            if (response.errorCode == BaseResponse.ERROR_CODE_SUCCESS) {
                if (isEmptyData(response.data)) {
                    response.dataState = DataState.STATE_EMPTY
                } else {
                    response.dataState = DataState.STATE_SUCCESS
                }
            } else {
                response.dataState = DataState.STATE_FAILED
// throw ServerResponseException(response.errorCode, response.errorMsg)
            }
        } catch (e: Exception) {
            response.dataState = DataState.STATE_ERROR
            response.exception = e
        } finally {
            responseLiveData.postValue(response)
            if (showLoading) {
                loadingStateLiveData.postValue(LoadingState(loadingMsg, DataState.STATE_FINISH))
            }
        }
    }
    
    private fun <T> isEmptyData(data: T?): Boolean {
        return data == null || data is List<*> && (data as List<*>).isEmpty()
    }
}
```

#### 2.3 ViewModel 层封装

如果不做封装，ViewModel层也会有模板代码。这里将通用代码在 BaseViewModel 中完成。BaseViewModel 需要持有 Repository。

```kotlin
abstract class BaseViewModel<T : BaseRepository> : ViewModel() {
    val repository: T by lazy {
        createRepository()
    }
    
    val loadingDataState: LiveData<LoadingState> by lazy {
        repository.loadingStateLiveData
    }
    
    /**
     * 创建Repository
     */
    @Suppress("UNCHECKED_CAST")
    open fun createRepository(): T {
        val baseRepository = findActualGenericsClass<T>(BaseRepository::class.java)
            ?: throw NullPointerException("Can not find a BaseRepository Generics in ${javaClass.simpleName}")
        return baseRepository.newInstance()
    }
}
```

#### 2.4 View 层封装

BaseActivity/BaseFragment 中持有 ViewModel，同时根据 LoadingState 处理 Loading 的显示与隐藏。

```kotlin
abstract class BaseActivity<VM : BaseViewModel<*>, VB : ViewBinding> : AppCompatActivity() {
    
    protected val viewModel by lazy {
        createViewModel()
    }
    
    protected val binding by lazy {
        createViewBinding()
    }
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(binding.root)
        viewModel.loadingDataState.observe(this) {
            when (it.state) {
                DataState.STATE_LOADING ->
                    showLoading(it.msg)
                else ->
                    dismissLoading()
            }
        }
        onActivityCreated(savedInstanceState)
    }
    
    /**
     * Activity content view created.
     *  @param  savedInstanceState savedInstanceState
     */
    abstract fun onActivityCreated(savedInstanceState: Bundle?)
    
    /**
     * 显示Loading
     */
    open fun showLoading(msg: String? = null) {
        ToastUtils.showToast("showLoading")
    }
    
    /**
     * 隐藏Loading
     */
    open fun dismissLoading() {
        ToastUtils.showToast("hideLoading")
    }
    
    /**
     * Create ViewBinding
     */
    @Suppress("UNCHECKED_CAST")
    open fun createViewBinding(): VB {
        val actualGenericsClass = findActualGenericsClass<VB>(ViewBinding::class.java)
            ?: throw NullPointerException("Can not find a ViewBinding Generics in ${javaClass.simpleName}")
        try {
            val inflate = actualGenericsClass.getDeclaredMethod("inflate", LayoutInflater::class.java)
            return inflate.invoke(null, layoutInflater) as VB
        } catch (e: NoSuchMethodException) {
            e.printStackTrace()
        } catch (e: IllegalAccessException) {
            e.printStackTrace()
        } catch (e: InvocationTargetException) {
            e.printStackTrace()
        }
    }
    
    /**
     * Create ViewModel
     *  @return  ViewModel
     */
    @Suppress("UNCHECKED_CAST")
    open fun createViewModel(): VM {
        val actualGenericsClass = findActualGenericsClass<VM>(BaseViewModel::class.java)
            ?: throw NullPointerException("Can not find a ViewModel Generics in ${javaClass.simpleName}")
        if (Modifier.isAbstract(actualGenericsClass.modifiers)) {
            throw IllegalStateException("$actualGenericsClass is an abstract class,abstract ViewModel class can not create a instance!")
        }
        // 判断如果是 BaseAndroidViewModel，则使用 AppViewModelFactory 来生成
        if (BaseAndroidViewModel::class.java.isAssignableFrom(actualGenericsClass)) {
            return ViewModelProvider(this, AppViewModelFactory(application))[actualGenericsClass]
        }
        return ViewModelProvider(this)[actualGenericsClass]
    }
}
```

创建 ViewModel 的过程是在 BaseActivity 中根据子类的泛型自动生成的。这里使用了反射的方式来实现。如果觉得影响性能可以在子类中重写`createViewModel`函数来自行生成
ViewModel。

另外，如果需要使用带有 Application 的ViewModel，可以继承 BaseAndroidViewModel,它的实现参照了 AndroidViewModel。

```kotlin
abstract class BaseAndroidViewModel<T : BaseRepository>(@field:SuppressLint("StaticFieldLeak") var application: Application) : BaseViewModel<T>()
```

这里创建 BaseAndroidViewModel 需要用到 AppViewModelFactory。

```kotlin
class AppViewModelFactory(private val application: Application) : ViewModelProvider.NewInstanceFactory() {
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        return if (BaseAndroidViewModel::class.java.isAssignableFrom(modelClass)) {
            try {
                modelClass.getConstructor(Application::class.java).newInstance(application)
            } catch (e: NoSuchMethodException) {
                throw IllegalStateException("Cannot create an instance of $modelClass", e)
            } catch (e: IllegalAccessException) {
                throw IllegalStateException("Cannot create an instance of $modelClass", e)
            } catch (e: InstantiationException) {
                throw IllegalStateException("Cannot create an instance of $modelClass", e)
            } catch (e: InvocationTargetException) {
                throw IllegalStateException("Cannot create an instance of $modelClass", e)
            }
        } else super.create(modelClass)
    }
}
```

假如 LoginViewModel 实现了 BaseAndroidViewModel ，那么使用 ViewModelProvider 创建 LoginViewModel 时必须传入
AppViewModelFactory 参数，而不是Jetpack ViewModel 库中的 AndroidViewModelFactory。

```kotlin
ViewModelProvider(this, AppViewModelFactory(application))[LoginViewModel::class.java]
```

### 3. MVVM 封装后的实例应用

在完成上述封装后，我们来看下如何实现登录的逻辑。

#### 3.1 实现 Repository

```kotlin
class LoginRepository : BaseRepository() {
    /**
     * Login
     *  @param  username 用户名
     *  @param  password 密码
     */
    suspend fun login(username: String, password: String, responseLiveData: ResponseMutableLiveData<LoginResponse>, showLoading: Boolean = true) {
        executeRequest({
            RetrofitManager.apiService.login(username, password)
        }, responseLiveData, showLoading)
    }
}
```

RetrofitManager 用来创建管理 API Service：

```kotlin
object RetrofitManager {
    private const val BASE_URL = "https://www.wanandroid.com/"
    
    val apiService: ApiService by lazy {
        RetrofitCreator.createApiService(
            ApiService::class.java, BASE_URL
        )
    }
}
```

#### 3.2 ViewModel 实例

```kotlin
class LoginViewModel : BaseViewModel<LoginRepository>() {
    // 提供给Model层设置数据
    private val _loginLiveData: ResponseMutableLiveData<LoginResponse> = ResponseMutableLiveData()
    
    // 提供给View层观察数据
    val loginLiveData: ResponseLiveData<LoginResponse> = _loginLiveData
    
    /**
     * Login
     *  @param  username 用户名
     *  @param  password 密码
     */
    fun login(username: String, password: String) {
        viewModelScope.launch {
            repository.login(username, password, _loginLiveData)
        }
    }
}
```

#### 3.3 Activity/Fragment 实例

```kotlin
class LoginActivity : BaseActivity<LoginViewModel, ActivityBizAMainBinding>() {
    
    override fun onActivityCreated(savedInstanceState: Bundle?) {
        binding.tvLogin.setOnClickListener {
            viewModel.login("110120", "123456")
        }
        
        viewModel.loginLiveData.observe(this, object : ResponseObserver<LoginResponse>() {
            override fun onSuccess(data: LoginResponse?) {
                // Update UI
            }
        })
    }
}
```

这里需要注意的是，在Activity里边需要使用不可变的 LiveData,防止开发时候在View层通过 LiveData
来setValue/postValue，造成错误的UI更新问题。因此，这里对LiveData做了一些改动,如下：

```kotlin
abstract class ResponseLiveData<T> : LiveData<BaseResponse<T>> {
    /**
     * Creates a MutableLiveData initialized with the given `value`.
     *
     *  @param  value initial value
     */
    constructor(value: BaseResponse<T>?) : super(value)
    
    /**
     * Creates a MutableLiveData with no value assigned to it.
     */
    constructor() : super()
    
    /**
     * Adds the given observer to the observers list within the lifespan of the given owner.
     * The events are dispatched on the main thread. If LiveData already has data set, it
     * will be delivered to the observer.
     */
    fun observe(owner: LifecycleOwner, observer: ResponseObserver<T>) {
        super.observe(owner, observer)
    }
}
```

ResponseLiveData 继承自 LiveData，因此ResponseLiveData是不可变的。除此之外还定义了一个 ResponseMutableLiveData ，这是一个可变的
LiveData，代码如下:

```kotlin
class ResponseMutableLiveData<T> : ResponseLiveData<T> {
    
    /**
     * Creates a MutableLiveData initialized with the given `value`.
     *
     *  @param  value initial value
     */
    constructor(value: BaseResponse<T>?) : super(value)
    
    /**
     * Creates a MutableLiveData with no value assigned to it.
     */
    constructor() : super()
    
    public override fun postValue(value: BaseResponse<T>?) {
        super.postValue(value)
    }
    
    public override fun setValue(value: BaseResponse<T>?) {
        super.setValue(value)
    }
}
```

至此关于Jetpack MVVM的封装及使用就结束了。

## 总结

本篇文章比较详细的讲解了Android项目架构从MVC、MVP到MVVM的演化，对三种架构都列举了详细的实现例子。同时针对目前主流的Jetpack
MVVM架构进行了封装。当然，由于每个人对于架构的理解并不一定相同，所有文章中避免不了会有与读者理解不一致的地方，欢迎留言讨论。



