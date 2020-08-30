---
title: RxJava+Retrofit完美封装（一）
date: 2017-04-30 03:29:43
tags: RxJava
---

***本篇文章已授权微信公众号 guolin_blog （郭霖）独家发布***

要说2016年最火的Android技术是什么，毫无疑问肯定是RxJava+Retrofit+Mvp。现如今2017年也已经过了快一半了。相信做android开发的小伙伴对RxJava和Retrofit也不再陌生。即使没有刻意的去学习过，也应该对RxJava和Retrofit有个一知半解。去年的时候学习了Rxjava和Retrofit的基本用法，但一直没有在实际项目中运用。今年开做新项目，果断在新项目中引入了RxJava和Retrofit。本篇文章将介绍笔者在项目中对Retrofit的封装。
先来看一下封装过后的Retrofit如何使用。
```
RetrofitHelper.getApiService()
                .getArticle()
                .compose(RxUtil.<ArticleWrapper>rxSchedulerHelper(this))
                .subscribe(new DefaultObserver<ArticleWrapper>() {
                    @Override
                    public void onSuccess(ArticleWrapper response) {
                        showToast("Request Success，size is：" + response.getDatas().size());
                    }
                });
```
没错，就是这么简洁的一个链式调用，可以显示加载动画，还加入了Retrofit生命周期的管理。
开始之前需要先在module项目里的Gradle文件中添加用到的依赖库
```
	compile "io.reactivex.rxjava2:rxjava:$rootProject.ext.rxjava2Version"
    compile "com.squareup.retrofit2:retrofit:$rootProject.ext.retrofit2Version"
    compile "com.squareup.retrofit2:converter-scalars:$rootProject.ext.retrofit2Version"
    compile "com.squareup.retrofit2:converter-gson:$rootProject.ext.retrofit2Version"
    compile "com.squareup.retrofit2:adapter-rxjava2:$rootProject.ext.retrofit2Version"
    compile 'com.jakewharton.retrofit:retrofit2-rxjava2-adapter:1.0.0'
    compile 'io.reactivex.rxjava2:rxandroid:2.0.1'
    compile 'com.squareup.okhttp3:logging-interceptor:3.4.1'
    compile "com.trello.rxlifecycle2:rxlifecycle:$rootProject.ext.rxlifecycle"
    //compile "com.trello.rxlifecycle2:rxlifecycle-android:$rootProject.ext.rxlifecycle"
    compile "com.trello.rxlifecycle2:rxlifecycle-components:$rootProject.ext.rxlifecycle"
```
为了方便依赖库版本的修改我们采用"io.reactivex.rxjava2:rxjava:$rootProject.ext.rxjava2Version"这中方式添加依赖，因此需要在project的build.gradle文件的加上以下内容：
```
ext {
    supportLibVersion = '25.1.0'
    butterknifeVersion = '8.5.1'
    rxjava2Version = '2.0.8'
    retrofit2Version = '2.2.0'
    rxlifecycle='2.1.0'
    gsonVersion = '2.8.0'
}
```
下面将通过几个小节对本次封装作详细的解析：

 - 服务器响应数据的基类BasicResponse
 - 构建初始化Retrofit的工具类IdeaApi
 - 通过GsonConverterFactory获取真实响应数据
 - 封装DefaultObserver处理服务器响应
 - 处理加载Loading
 - 管理Retrofit生命周期
 - 如何使用封装
 - 小结

一.服务器响应数据的基类BasicResponse。
----------------------------

假定服务器返回的Json数据格式如下：
```
{
 "code": 200,
 "message": "成功",
 "content": {
	...
	}
}
```
根据Json数据格式构建我们的BasicResponse（BasicResponse中的字段内容需要根据自己服务器返回的数据确定）。代码如下：

```
public class BasicResponse<T> {

    private int code;
    private String message;
    private T content;
	...此处省去get、set方法。
```

二.构建初始化Retrofit的工具类IdeaApi。
---------------------------

该类通过RetrofitUtils来获取ApiService的实例。代码如下：
```
public class IdeaApi {
    public static <T> T getApiService(Class<T> cls,String baseUrl) {
        Retrofit retrofit = RetrofitUtils .getRetrofitBuilder(baseUrl).build();
        return retrofit.create(cls);
    }
}
```
RetrofitUtils用来构建Retrofit.Builder，并对OkHttp做以下几个方面的配置：
 1.  设置日志拦截器，拦截服务器返回的json数据。Retrofit将请求到json数据直接转换成了实体类，但有时候我们需要查看json数据，Retrofit并没有提供直接获取json数据的功能。因此我们需要自定义一个日志拦截器拦截json数据，并输入到控制台。
 2. 设置Http请求头。给OkHttp 添加请求头拦截器，配置请求头信息。还可以为接口统一添加请求头数据。例如，把用户名、密码（或者token）统一添加到请求头。后续每个接口的请求头中都会携带用户名、密码（或者token）数据，避免了为每个接口单独添加。
 3. 为OkHttp配置缓存。同样可以同过拦截器实现缓存处理。包括控制缓存的最大生命值，控制缓存的过期时间。
 4. 如果采用https，我们还可以在此处理证书校验以及服务器校验。
 5. 为Retrofit添加GsonConverterFactory。此处是一个比较重要的环节，将在后边详细讲解。
RetrofitUtils 代码如下：
```
public class RetrofitUtils {
    public static OkHttpClient.Builder getOkHttpClientBuilder() {

        HttpLoggingInterceptor loggingInterceptor = new HttpLoggingInterceptor(new HttpLoggingInterceptor.Logger() {
            @Override
            public void log(String message) {
                try {
                    LogUtils.e("OKHttp-----", URLDecoder.decode(message, "utf-8"));
                } catch (UnsupportedEncodingException e) {
                    e.printStackTrace();
                    LogUtils.e("OKHttp-----", message);
                }
            }
        });
        loggingInterceptor.setLevel(HttpLoggingInterceptor.Level.BODY);

        File cacheFile = new File(Utils.getContext().getCacheDir(), "cache");
        Cache cache = new Cache(cacheFile, 1024 * 1024 * 100); //100Mb

        return new OkHttpClient.Builder()
                .readTimeout(Constants.DEFAULT_TIMEOUT, TimeUnit.MILLISECONDS)
                .connectTimeout(Constants.DEFAULT_TIMEOUT, TimeUnit.MILLISECONDS)
                .addInterceptor(loggingInterceptor)
                .addInterceptor(new HttpHeaderInterceptor())
                .addNetworkInterceptor(new HttpCacheInterceptor())
               // .sslSocketFactory(SslContextFactory.getSSLSocketFactoryForTwoWay())  // https认证 如果要使用https且为自定义证书 可以去掉这两行注释，并自行配制证书。
               // .hostnameVerifier(new SafeHostnameVerifier())
                .cache(cache);
    }

    public static Retrofit.Builder getRetrofitBuilder(String baseUrl) {
        Gson gson = new GsonBuilder().setDateFormat("yyyy-MM-dd HH:mm:ss").serializeNulls().create();
        OkHttpClient okHttpClient = RetrofitUtils.getOkHttpClientBuilder().build();
        return new Retrofit.Builder()
                .client(okHttpClient)
                .addConverterFactory(GsonConverterFactory.create(gson))
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .baseUrl(baseUrl);
    }
}
```

三.通过GsonConverterFactory获取真实响应数据
--------------------------------
在第一节中我们构建了服务器响应数据BasicResponse，BasicResponse由code、message、和content三个字段。其中code为服务器返回的错误码。我们会事先和服务器约定成功时的code值，比如200表示请求成功。但通常在请求服务器数据过程中免不了会出现各种错误。例如用户登录时密码错误、请求参数错误的情况。此时服务器会根据错误情况返回对应的错误码。一般来说，我们只关心成功时即code为200时的content数据。而对于code不为200时我们只需要给出对应的Toast提示即可。事实上我们对我们有用的仅仅时code为200时的content数据。因此我们可以考虑过滤掉code和message，在请求成功的回调中只返回content的内容。
在此种情况下就需要我们通过自定义GsonConverterFactory来实现了。我们可以直接从Retrofit的源码中copy出GsonConverterFactory的三个相关类来做修改。
其中最终要的一部分是修改GsonResponseBodyConverter中的convert方法。在该方法中拿到服务器响应数据并判断code是否为200。如果是，则获取到content并返回，如果不是，则在此处可以抛出对应的自定义的异常。然后再Observer中统一处理异常情况。GsonResponseBodyConverter代码如下：
```
final class GsonResponseBodyConverter<T> implements Converter<ResponseBody, Object> {

    private final TypeAdapter<T> adapter;

    GsonResponseBodyConverter(TypeAdapter<T> adapter) {
        this.adapter = adapter;
    }

    @Override
    public Object convert(ResponseBody value) throws IOException {
        try {
            BasicResponse response = (BasicResponse) adapter.fromJson(value.charStream());
            if (response.getCode()==200) {
            return response.getResults();
            } else {
                // 特定 API 的错误，在相应的 DefaultObserver 的 onError 的方法中进行处理
                throw new ServerResponseException(response.getCode(), response.getMessage());
            }
        } finally {
            value.close();
        }
        return null;
    }
}
```

四.构建DefaultObserver处理服务器响应。
-----------------------------
上一节中我们讲到了在请求服务器时可能出现的一些例如密码错误、参数错误的情况，服务器给我们返回了对应的错误码，我们根据错误码抛出了对应自定义异常。除此之外在我们发起网络请求时还可能发生一些异常情况。例如没有网络、请求超时或者服务器返回了数据但在解析时出现了数据解析异常等。对于这样的情况我们也要进行统一处理的。那么我们就需要自定义一个DefaultObserver类继承Observer，并重写相应的方法。
该类中最重要的两个方法时onNext和onError。
**1.在服务器返回数据成功的情况下会回调到onNext方法。**因此我们可以在DefaultObserver中定义一个抽象方法onSuccess(T response)，在调用网络时重写onSuccess方法即可。
**2.如果在请求服务器过程中出现任何异常，都会回调到onError方法中。**包括上节中我们自己抛出的异常都会回调到onError。因此我们的重头戏就是处理onError。在onError中我们根据异常信息给出对应的Toast提示即可。
DefaultObserver类的代码如下：
```
public abstract class DefaultObserver<T> implements Observer<T> {
    @Override
    public void onSubscribe(Disposable d) {

    }

    @Override
    public void onNext(T response) {
        onSuccess(response);
        onFinish();
    }

    @Override
    public void onError(Throwable e) {
        LogUtils.e("Retrofit", e.getMessage());
        if (e instanceof HttpException) {     //   HTTP错误
            onException(ExceptionReason.BAD_NETWORK);
        } else if (e instanceof ConnectException
                || e instanceof UnknownHostException) {   //   连接错误
            onException(ExceptionReason.CONNECT_ERROR);
        } else if (e instanceof InterruptedIOException) {   //  连接超时
            onException(ExceptionReason.CONNECT_TIMEOUT);
        } else if (e instanceof JsonParseException
                || e instanceof JSONException
                || e instanceof ParseException) {   //  解析错误
            onException(ExceptionReason.PARSE_ERROR);
        }else if(e instanceof ServerResponseException){
            onFail(e.getMessage());
        } else {
            onException(ExceptionReason.UNKNOWN_ERROR);
        }
        onFinish();
    }

    @Override
    public void onComplete() {
    }

    /**
     * 请求成功
     *
     * @param response 服务器返回的数据
     */
    abstract public void onSuccess(T response);

    /**
     * 服务器返回数据，但响应码不为200
     *
     */
    public void onFail(String message) {
        ToastUtils.show(message);
    }
    
    public void onFinish(){}

    /**
     * 请求异常
     *
     * @param reason
     */
    public void onException(ExceptionReason reason) {
        switch (reason) {
            case CONNECT_ERROR:
                ToastUtils.show(R.string.connect_error, Toast.LENGTH_SHORT);
                break;

            case CONNECT_TIMEOUT:
                ToastUtils.show(R.string.connect_timeout, Toast.LENGTH_SHORT);
                break;

            case BAD_NETWORK:
                ToastUtils.show(R.string.bad_network, Toast.LENGTH_SHORT);
                break;

            case PARSE_ERROR:
                ToastUtils.show(R.string.parse_error, Toast.LENGTH_SHORT);
                break;

            case UNKNOWN_ERROR:
            default:
                ToastUtils.show(R.string.unknown_error, Toast.LENGTH_SHORT);
                break;
        }
    }

    /**
     * 请求网络失败原因
     */
    public enum ExceptionReason {
        /**
         * 解析数据失败
         */
        PARSE_ERROR,
        /**
         * 网络问题
         */
        BAD_NETWORK,
        /**
         * 连接错误
         */
        CONNECT_ERROR,
        /**
         * 连接超时
         */
        CONNECT_TIMEOUT,
        /**
         * 未知错误
         */
        UNKNOWN_ERROR,
    }
}
```

五.处理加载Loading
----------------------
关于Loading我们可以通过RxJava的compose操作符来做一个非常优雅的处理。首先定义一个ProgressUtils工具类，然后通过RxJava的ObservableTransformer做一个变换来处理Loading。想要显示Loading，只需要加上.compose(ProgressUtils.< T >applyProgressBar(this))即可。
ProgressUtils代码如下：
```
public class ProgressUtils {
    public static <T> ObservableTransformer<T, T> applyProgressBar(
            @NonNull final Activity activity, String msg) {
        final WeakReference<Activity> activityWeakReference = new WeakReference<>(activity);
        final DialogUtils dialogUtils = new DialogUtils();
        dialogUtils.showProgress(activityWeakReference.get());
        return new ObservableTransformer<T, T>() {
            @Override
            public ObservableSource<T> apply(Observable<T> upstream) {
                return upstream.doOnSubscribe(new Consumer<Disposable>() {
                    @Override
                    public void accept(Disposable disposable) throws Exception {

                    }
                }).doOnTerminate(new Action() {
                    @Override
                    public void run() throws Exception {
                        Activity context;
                        if ((context = activityWeakReference.get()) != null
                                && !context.isFinishing()) {
                            dialogUtils.dismissProgress();
                        }
                    }
                }).doOnSubscribe(new Consumer<Disposable>() {
                    @Override
                    public void accept(Disposable disposable) throws Exception {
                        /*Activity context;
                        if ((context = activityWeakReference.get()) != null
                                && !context.isFinishing()) {
                            dialogUtils.dismissProgress();
                        }*/
                    }
                });
            }
        };
    }

    public static <T> ObservableTransformer<T, T> applyProgressBar(
            @NonNull final Activity activity) {
        return applyProgressBar(activity, "");
    }
}
```
至此关于RxJava和Retrofit的二次封装已经基本完成。但是我们不能忽略了很重要的一点，就是网络请求的生命周期。我们将在下一节中详细讲解。

六、管理Retrofit生命周期
----------------
当activity被销毁时，网络请求也应该随之终止的。要不然就可能造成内存泄漏。会严重影到响App的性能！因此Retrofit生命周期的管理也是比较重要的一点内容。在这里我们使用 **[RxLifecycle](https://github.com/trello/RxLifecycle)**来对Retrofit进行生命周期管理。其使用流程如下：

**1.在gradel中添加依赖如下：**
```
compile 'com.trello.rxlifecycle2:rxlifecycle:2.1.0'
compile 'com.trello.rxlifecycle2:rxlifecycle-components:2.1.0'

```
**2.让我们的BaseActivity继承RxAppCompatActivity。**
具体代码如下：
```
public abstract class BaseActivity extends RxAppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(getLayoutId());
        init(savedInstanceState);
    }
    protected void showToast(String msg) {
        ToastUtils.show(msg);
    }

    protected abstract @LayoutRes int getLayoutId();

    protected abstract void init(Bundle savedInstanceState);
}
```
同样我们项目的BaseFragment继承RxFragment（注意使用继承V4包下的RxFragment），如下：
```
public abstract class BaseFragment extends RxFragment {

    public View rootView;
    public LayoutInflater inflater;


    @Nullable
    @Override
    public final View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
        super.onCreateView(inflater, container, savedInstanceState);
        this.inflater = inflater;
        if (rootView == null) {
            rootView = inflater.inflate(this.getLayoutId(), container, false);
            init(savedInstanceState);
        }
        ViewGroup parent = (ViewGroup) rootView.getParent();
        if (parent != null) {
            parent.removeView(rootView);
        }
        return rootView;
    }

    protected abstract int getLayoutId();

    protected abstract void init(Bundle savedInstanceState);

    protected void showToast(String msg) {
        ToastUtils.show(msg);
    }

    @Override
    public void onResume() {
        super.onResume();
    }

    @Override
    public void onPause() {
        super.onPause();
    }


    @Override
    public void onDestroyView() {
        super.onDestroyView();
    }
}
```
3.使用compose操作符管理Retrofit生命周期了:

```
myObservable
            .compose(bindToLifecycle())
            .subscribe();

或者

myObservable
    .compose(RxLifecycle.bindUntilEvent(lifecycle, ActivityEvent.DESTROY))
    .subscribe();

```
关于RxLifecycle的详细使用方法可以参考 **[RxLifecycle官网](https://github.com/trello/RxLifecycle)**

七.如何使用封装
--------
前面几节内容讲解了如何RxJava进行二次封装，封装部分的代码可以放在我们项目的Library模块中。那么封装好之后我们应该如何在app模块中使用呢？
**1.定义一个接口来存放我们项目的API**
```
public interface IdeaApiService {
    /**
     * 此接口服务器响应数据BasicResponse的泛型T应该是List<MeiZi>
     * 即BasicResponse<List<MeiZi>>
     * @return BasicResponse<List<MeiZi>>
     */
    @Headers("Cache-Control: public, max-age=10")//设置缓存 缓存时间为100s
    @GET("福利/10/1")
    Observable<List<MeiZi>> getMezi();

    /**
     * 登录 接口为假接口 并不能返回数据
     * @return
     */
    @POST("login.do")
    Observable<LoginResponse> login(@Body LoginRequest request);

    /**
     * 刷新token 接口为假接口 并不能返回数据
     * @return
     */
    @POST("refresh_token.do")
    Observable<RefreshTokenResponseBean> refreshToken(@Body RefreshTokenRequest request);

    @Multipart
    @POST("upload/uploadFile.do")
    Observable<BasicResponse> uploadFiles(@Part List<MultipartBody.Part> partList);
}
```

**2.定义一个RetrofitHelper 类，通过IdeaApi来获取IdeaApiService的实例。**
```
public class RetrofitHelper {
    private static IdeaApiService mIdeaApiService;

    public static IdeaApiService getApiService(){
        return mIdeaApiService;
    }
    static {
       mIdeaApiService= IdeaApi.getApiService(IdeaApiService.class, Constants.API_SERVER_URL);
    }
}
```
**3.在Activity或者Fragment中发起网络请求**

```
    /**
     * Get请求
     * @param view
     */
    public void getData(View view) {
        RetrofitHelper.getApiService()
                .getMezi()
                .compose(this.<List<MeiZi>>bindToLifecycle())
                .compose(ProgressUtils.<List<MeiZi>>applyProgressBar(this))
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new DefaultObserver<List<MeiZi>>() {
                    @Override
                    public void onSuccess(List<MeiZi> response) {
                        showToast("请求成功，妹子个数为" + response.size());
                    }
                });
    }
```

八.小结
----
本篇文章主要讲解了Rxjava和Retrofit的二次封装。以上内容也是笔者参考多方面的资料经过长时间的改动优化而来。但鉴于本人能力有限，其中也避免不了出现不当之处。还请大家多多包涵。另外，在投稿郭神公众号时文章可能还存在很多处理不优雅的地方，比如对响应数据的处理以及对Loading的处理。在投稿被推送后收到了很多小伙伴的建议，因此笔者也参考了大家的意见并做了优化，在此感谢大家。最后如果有疑问欢迎在文章留言评论。
													 
[（二）Rxjava2+Retrofit之Token自动刷新](https://blog.csdn.net/qq_20521573/article/details/76100558)
[（三）Rxjava2+Retrofit实现文件上传与下载](https://blog.csdn.net/qq_20521573/article/details/78356747)

[ 源码传送门](https://github.com/zhpanvip/Retrofit2)

