---
title: 使用Rxjava+Retrofit+Dagger2+Mvp构建的安卓框架（1）
date: 2017-02-05 19:49:20
updated: 2017-02-05 19:49:20
categories: [项目架构]
tags: [项目架构]
---
# 网络请求搭建
>采用这种方式搭建的项目架构，比较的解耦，要是追求更高的解耦，也可以加上mvvm，有时间的话会加上mvvm的文章。Rxjava对线程的处理非常好，
MVP设计模式把业务处理和用户交互分开，P负责处理业务，V负责处理和用户的交互，以及界面的展示，Dagger2的好处前面也已经说过了，就是解耦。

>这个系列的第一篇文章主要写怎么打通网络请求这一块。

## 包的结构。
>框架的搭建主要设计分为那些模块。我们新建一个data module用于处理网络请求以及数据。项目的整体包结构是下面这个样子。
<img src="https://i.niupic.com/images/2017/02/07/rORcUD.png" />
#### java
1. base 一些基类。
2. di dagger2相关的类。
3. ui 各模块的类。
3. main MainActivity相关的类，一般一个模块，我们就给他建一个包。
#### data
1. api 网络接口相关的api
2. bean 解析网络数据的实体
3. repository 网络请求的仓库。

## 网络请求模块如何搭建起来。

* 网络接口api
```java
public interface DailyApi {

    //获取最新日报新闻列表
    String URL_GET_LATEST_NEWS = "news/latest";

    /**
     * 获取今日日报新闻列表
     *
     * @return TodayNews
     */
    @GET(URL_GET_LATEST_NEWS)
    Observable<NewsBean> getTodayNews();

}
```

* 网络请求仓库
```java
public class DailyRepository {

    DailyApi mDailyApi;

    public DailyRepository(DailyApi dailyApi) {
        mDailyApi = dailyApi;
    }

    public Observable<NewsBean> getNewsData() {
        return mDailyApi.getTodayNews();
    }
}
```
* 网络请求需要的一些类，都由module提供。
```java
/**
 * @创建者 frank
 * @时间 2017/2/6 15:39
 * @描述：${提供和网络相关的类对象}
 */
@Module
public class DataModule {

    private static final String BASE_URL = "http://news-at.zhihu.com/api/4/";
    
    @Singleton
    @Provides
    public DailyRepository provideDailyRepository(DailyApi dailyApi) {
        return new DailyRepository(dailyApi);
    }


    @Singleton
    @Provides
    public DailyApi provideDailyApi(Retrofit retrofit) {
        return retrofit.create(DailyApi.class);
    }


    /**
     * Retrofit
     *
     * @param client           OkHttpClient
     * @param converterFactory Converter.Factory
     * @return Retrofit
     */
    @Provides
    @Singleton
    public Retrofit provideClientApi(OkHttpClient client, Converter.Factory converterFactory) {
        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(BASE_URL)
                .addConverterFactory(converterFactory)
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                .client(client)
                .build();
        return retrofit;
    }


    /**
     * Gson转换器单例对象
     *
     * @param gson Gson
     * @return Converter.Factory
     */
    @Provides
    @Singleton
    public Converter.Factory provideConverter(Gson gson) {
        return GsonConverterFactory.create(gson);
    }


    /**
     * Gson 单例对象
     *
     * @return Gson
     */
    @Provides
    @Singleton
    public Gson provideGson() {
        return new GsonBuilder().serializeNulls().create();
    }

    /**
     * OkHttp客户端单例对象
     *
     * @param loggingInterceptor HttpLoggingInterceptor
     * @return OkHttpClient
     */
    @Provides
    @Singleton
    public OkHttpClient provideClient(HttpLoggingInterceptor loggingInterceptor) {
        OkHttpClient client = new OkHttpClient.Builder()
                .addInterceptor(loggingInterceptor)
                .addNetworkInterceptor(new StethoInterceptor())
                .build();
        return client;
    }

    /**
     * 日志拦截器单例对象,用于OkHttp层对日志进行处理
     *
     * @return HttpLoggingInterceptor
     */
    @Provides
    @Singleton
    public HttpLoggingInterceptor provideLogger() {
        HttpLoggingInterceptor interceptor = new HttpLoggingInterceptor();
        interceptor.setLevel(HttpLoggingInterceptor.Level.NONE);
        return interceptor;
    }
    
}
```

* 网络请求统一的管理类，
```java
public class DataManager {
    @Inject
    public DailyRepository mDailyRepository;
    public DataManager() {
        DaggerDataComponent.builder().dataModule(new DataModule()).build().injectDataManager(this);
    }
}

```
* 在AppModule中提供DataManager

```java
@Module
public class AppModule {
    Application mApplication;
    public AppModule(Application application) {
        mApplication = application;
    }

    @Singleton
    @Provides
    Application provideApplication(){
        return  mApplication;
    }

    @Singleton
    @Provides
    DataManager provideDataManager(){

        return new DataManager();
    }
}
```

```java
@Singleton
@Component(modules = AppModule.class)
public interface AppComponent {

    void injectApp(App app);

    Application getApplication();

    DataManager getDataManager();

}

```

* 全局的application
```java
public class App extends Application {

    @Inject
    static DataManager mDataManager;
    
    AppComponent mAppComponent;
    
    @Override
    public void onCreate() {
        super.onCreate();

        mAppComponent = DaggerAppComponent.builder().appModule(new AppModule(this)).build();
        mAppComponent.injectApp(this);

        if (BuildConfig.DEBUG) {
            Stetho.initialize(Stetho.newInitializerBuilder(this)
                    .enableDumpapp(Stetho.defaultDumperPluginsProvider(this))
                    .enableWebKitInspector(Stetho.defaultInspectorModulesProvider(this))
                    .build());
        }
    }

    public AppComponent getAppComponent() {
        return mAppComponent;
    }

    public static DataManager getDataManager() {
        return mDataManager;
    }

}
```

* 最后在presenter中获取到DataManager,然后进行网络请求。

```java
public class BasePresenter<V extends IView> implements IPresenter<V> {

    protected V mView;
    protected DataManager mDataManager;
    private CompositeSubscription mCompositeSubscription;

    public BasePresenter() {
        mDataManager = App.getDataManager();
    }
}
```

>整个的流程是这样：在Application中注入DataManager,DataManager中拥有各个Repository的成员，然后Repository是通过DataComponent
注入到DataManager中的，关于网络请求需要用到的其他类，比如Retrofit,OkHttpClient,Api等，都是通过DataModule提供的，然后在presenter
里面拿到DataManager，就可以进行网络请求了。

[源代码已经上传到github](https://github.com/frankandroid/FrameWorkSample);
