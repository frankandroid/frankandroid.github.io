---
title: 关于dagger2及其使用(2)haha
date: 2017-02-04 10:05:32
updated: 2017-02-04 10:05:32
categories:
tags:
---

>接着上一篇的博客，我们来接着了解dagger2的其他操作符及其使用。包括Qualifier（限定符）、Singleton（单例）、Scope（作用域）,SubComponent等。

# Qualifier

>Qualifier 这是一个限定符注解，什么情况下我们会需要用到限定符注解呢？当一个类的实例，有多种提供方法时，比如module里面提供了多个FirstStudent的实例
如果不使用限定符，那么dagger2会不知道究竟应该注入哪个，就会报错。接下来看在代码中如何具体的使用。

>首先在module提供类实例的方法上加上限定符，@Named是Dagger2对于@Qualifier一个默认实现
```java
/**
 * @创建者 frank
 * @时间 2017/1/22 17:45
 * @描述：${module提供类的实例，当对于同一种类的实例，有多种提供方式时，需要用到@Qualifier限定符}
 */
@Module
public class FirstModule {
    /**
     *当一个类的实例有多种方式可以提供时，那么dagger2不知道需要去注入哪一个类的实例，这个时候，就需要用到@Qualifier限定符。
     * @Named是Dagger2对于@Qualifier一个默认实现，我们也可以自定义，比如@ForApplication和@ForAcitivity来标识不同的Context
     * */
    @Named("agedStudent")
    @Provides
    FirstStudent provideAgedStudent(){
        return new FirstStudent(40);
    }

    @Named("namedStudent")
    @Provides
    FirstStudent provideNamedStudent(){
        return new FirstStudent("谢凯");
    }
}
```
>接下来，在注入的类中，添加限定符。
```java
    @Inject
    @Named("namedStudent")
    //@ForActivity
    FirstStudent mFirstStudent;
```

>如果需要自定义限定符。
```java
@Qualifier
@Documented
@Retention(RUNTIME)
public @interface ForActivity {
}

```

# Scope
>Scope是一个注解作用域，通过自定义注解限定对象的作用范围。通过这个注解能够解决不同对象生命周期不一致的问题.singleton是dagger自带的scope。
我们如果需要创建全局的单例</br>
* 在Module中定义创建全局类实例的方法
* ApplicationComponent管理Module
* 保证ApplicationComponent只有一个实例（在app的Application中实例化ApplicationComponent）

>singleton其实就是dagger2默认提供的一个注解。和我们自己定义的scope注解并没有区别，如果你喜欢，也可以用以下的注解代替sigleton.
```java
@Scope
@Retention(RetentionPolicy.RUNTIME)
public @interface PerApp {

}
```
>比如我们要创建一个全局单例的context
1. 在Module中定义创建全局类实例的方法
```java
@Module
public class AppModule {

    Context mContext;

    public AppModule(Context context) {
        mContext = context;
    }

    @Provides
    @PerApp
    Context provideAppContext() {
        return mContext;
    }
}
```


2. ApplicationComponent管理Module
```java
@PerApp
@Component(modules = AppModule.class)
public interface AppComponent {

    Context getAppContext();

}
```
3. 保证ApplicationComponent只有一个实例（在app的Application中实例化ApplicationComponent）

```java
public class App extends Application {
    AppComponent mAppComponent;
    @Override
    public void onCreate() {
        super.onCreate();
        mAppComponent = DaggerAppComponent.builder().appModule(new AppModule(this)).build();
    }
}
```

>到这里我们就介绍了Scope和Qualifier，接下来我们跟着component之间的组织方式，了解其他的注解。

# Component的组织方式。

>Component的组织方式可以解决类实例共享的问题，比如其他的Component想要把全局的类实例注入到目标类中。具体的组织方式有以下三种。

### 依赖方式

一个Component是依赖于一个或多个Component，Component中的dependencies属性就是依赖方式的具体实现，例如，ActivityComponent依赖AppComponent

### 包含方式
一个Component是包含一个或多个Component的，被包含的Component还可以继续包含其他的Component。这种方式特别像Activity与Fragment的关系。SubComponent就是包含方式的具体实现。

### 继承方式
官网没有提到该方式，具体没有提到的原因我觉得应该是，该方式不是解决类实例共享的问题，而是从更好的管理、维护Component的角度，把一些Component共有的方法抽象到一个父类中，然后子Component继承


#### 需要注意的细节
1. 就是MainComponent继承ActivityComponent,那么也要标明作用域。@PerActivity

2. Component可以注入通过injectXXX()方法直接注入一个类，也可以通过返回一个类的对象，直接对外提供类的实例。看下面的代码。

3. MainComponent依赖于AppComponent,并不是说AppModule中有的就可以注入，而是要AppComponent已经注入过的。例如

```java
@Module
public class AppModule {

    Context mContext;

    public AppModule(Context context) {
        mContext = context;
    }

    @Provides
    @PerApp
    Context provideAppContext() {
        return mContext;
    }

    @Named("namedStudent")
    @Provides
     FirstStudent provideNamedStudent(){
        return new FirstStudent("谢凯");
    }
    
}
```

```java
@Module
public class MainModule {

    @Named("agedStudent")
    @Provides
    FirstStudent provideAgedStudent(){
        return new FirstStudent(40);
    }

   /* @Named("namedStudent")
    @Provides
    FirstStudent provideNamedStudent(){
        return new FirstStudent("谢凯");
    }*/

}
```
>以上的代码编译会报错，FirstStudent注入失败，原因是因为，依赖是指能够使用提供的实例，而不是指没有提供的，module里面有的。（上面的代码中，虽然AppModule中有提供FirstStudent,但是，在appComponent中并没有一个方法返回FirstStudent，所以当运行的时候会报错，因为
MainActivity中需要注入FirstStudent，但是在MainModule中找不到，并且，在AppComponent中也没有提供，所以当然就报错。）所以正确的做法是下面这样子的。

1. MainComponent依赖于AppComponent
```java
@PerActivity
@Component(dependencies = AppComponent.class,modules = {ActivityModule.class, MainModule.class})
public interface MainComponent extends ActivityComponent{

    void injectActivity(MainActivity mainActivity);

    MainFragmentComponent getMainFragmentComponent();

}
```
2. AppComponent中提供加了@Named注解的FirstStudent。
```java
@PerApp
@Component(modules = AppModule.class)
public interface AppComponent {

    Context getAppContext();

    @Named("agedStudent")
    FirstStudent provideFirstStudent();

}
```

3. AppModule中提供加了@Named注解的FirstStudent。
```java
@Module
public class AppModule {

    Context mContext;

    public AppModule(Context context) {
        mContext = context;
    }

    @Provides
    @PerApp
    Context provideAppContext() {
        return mContext;
    }

    @Named("agedStudent")
    @Provides
    FirstStudent provideAgedStudent(){
        return new FirstStudent(40);
    }

}
```
>这个时候，MainComponent就可以提供一个FirstStudent,当在MainActivity中注入了FirstStudent就能够注入成功。
```java
public class MainActivity extends BaseActivity implements View.OnClickListener {

    private String mStudentJson = "{\n" +
            "    \"name\": \"Jack\",\n" +
            "    \"age\": 18,\n" +
            "    \"isBoy\": true\n" +
            "}";

    @Inject
    @Named("namedStudent")
    FirstStudent mFirstStudent;

    @Inject
    @Named("agedStudent")
    FirstStudent mFirstAgeStudent;

    private FirstComponent mFirstComponent;
    private ThirdLibComponent mThirdLibComponent;
    private TextView mShowContent;
    private GsonTestStudent mGsonTestStudent;

    private MainComponent mMainComponent;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_scrolling);
        Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);

        mMainComponent = DaggerMainComponent.builder()
                .mainModule(new MainModule())
                .activityModule(new ActivityModule())
                .appComponent(((App)getApplication()).getAppComponent())
                .build();

        mMainComponent.injectActivity(this);

        mGsonTestStudent = mGson.fromJson(mStudentJson, GsonTestStudent.class);

        FloatingActionButton fab = (FloatingActionButton) findViewById(R.id.fab);
        fab.setOnClickListener(this);

        Button injectFirst = (Button) findViewById(R.id.inject_first_student);
        injectFirst.setOnClickListener(this);

        Button injectModule = (Button) findViewById(R.id.inject_with_module);
        injectModule.setOnClickListener(this);

        mShowContent = (TextView) findViewById(R.id.show_content);

    }

    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.fab:
                Snackbar.make(v, "dagger2", Snackbar.LENGTH_LONG)
                        .setAction("Action", null).show();
                break;
            case R.id.inject_first_student:
                mShowContent.setText("");
                break;
            case R.id.inject_with_module:
                mShowContent.setText(mGsonTestStudent.toString() + "---"
                        + mFirstStudent.toString()
                        +"---"+mFirstAgeStudent.toString());
                break;
        }
    }
}
```

# Subcomponent 
>表明自己是被包含的Component,和dependencies类似。就是上级提供的你都有。
```java
@PerActivity
@Subcomponent
public interface MainFragmentComponent {

    void injectFragment(MainFragment mainFragment);

}

```

```java
@PerActivity
@Component(dependencies = AppComponent.class, modules = {ActivityModule.class, MainModule.class})
public interface MainComponent extends ActivityComponent {

    void injectActivity(MainActivity mainActivity);

    MainFragmentComponent getMainFragmentComponent();

    @Named("namedStudent")
    FirstStudent provideFirstStudent();

}
```

```java

    @Override
    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);

        if (getActivity() instanceof MainActivity) {
           mMainFragmentComponent = ((MainActivity) getActivity()).getMainComponent().getMainFragmentComponent();
           mMainFragmentComponent.injectFragment(this);
           mTextView.setText(mFirstStudent.toString());
        }
    }

```

>大概就是这么多了。





