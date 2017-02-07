---
title: 使用Rxjava-Retrofit-Dagger2-Mvp构建的安卓框架(2)
date: 2017-02-07 20:28:13
updated: 2017-02-07 20:28:13
categories: [项目架构]
tags: [项目架构]
---

# MVP搭建
>接着上一篇，这篇主要聊如何组织MVP中的M,V,P

直接上代码，考虑到Presenter获取到View的数据之后，一般也不会有太多复杂的处理。所以就没有另外加入Modle.加上也不复杂。

### 1. Presenter 和 View 的基类，Presenter要拥有View的实例对象，才能够调用View的方法。所以提供两个必须实现的方法。一个是让
Presenter用于View的实例，另一个是为了在View销毁的时候，释放Presenter,避免内存泄漏。

```java
public interface IPresenter<V extends IView> {

    void onAttachView(V view);

    void onDetachView(V view);
    
}
```

```java

public interface IView {
    
}
```
### 2. MvpActivity，MvpView的基类。获取Presenter的实例，销毁的时候，presenter作相应处理，避免内存泄漏。

```java
public abstract class MvpActivity<P extends IPresenter> extends BaseActivity implements IView {

    protected P mPresenter;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mPresenter = initPresenter();
    }

    /**
     * 由子类去创建presenter
     */
    protected abstract P initPresenter();


    @Override
    public void onDestroy() {
        super.onDestroy();
        if (mPresenter != null)
            mPresenter.onDetachView(this);
    }

}

```

### 3. BasaPresenter, Presenter的基类。View销毁的时候取消RxJava的注册。获取到DataManager和View对象。
```java
public abstract class BasePresenter<V extends IView> implements IPresenter<V> {

    protected V mView;
    protected DataManager mDataManager;
    private CompositeSubscription mCompositeSubscription;

    public BasePresenter(V view) {
        mDataManager = App.getInstance().getDataManager();
        onAttachView(view);
    }

    @Override
    public void onAttachView(V view) {
      this.mView = view;
    }

    @Override
    public void onDetachView(IView view) {
        onUnSubscribe();
    }
    /**
     * rxJava取消注册，以避免内存泄露
     */
    protected void onUnSubscribe() {
        if (mCompositeSubscription != null && mCompositeSubscription.hasSubscriptions()) {
            mCompositeSubscription.unsubscribe();
        }
    }

    /**这里是统一处理observable,这样就不用在每次网络请求回来后都加上线程切换的代码*/
    public <T> void addSubscription(Observable<T> observable, Subscriber<T> subscriber) {
        if (mCompositeSubscription == null) {
            mCompositeSubscription = new CompositeSubscription();
        }
        Subscription subscription = observable
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(subscriber);
        mCompositeSubscription.add(subscription);
    }
}
```

### 4. 具体的View和Presenter的实例。
#### 沟通的桥梁
```java
public class MainContact {

    interface View extends IView {

        void onGetDailyDataSuc(NewsBean newsBean);
        void onGetDailyDataFailed(String msg);

    }

    interface Presenter extends IPresenter<View> {

       void GetDailyData();

    }
}
```
#### MainActivity
```java
public class MainActivity extends MvpActivity<MainContact.Presenter> implements MainContact.View {

    @BindView(R.id.bt)
    Button mBt;
    @BindView(R.id.tv)
    TextView mTv;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.bind(this);

    }

    @Override
    protected MainContact.Presenter initPresenter() {
        return new MainPresenter(this);
    }

    @OnClick({R.id.bt})
    void onButtonClick(View view) {
        switch (view.getId()) {
            case R.id.bt:
                mPresenter.GetDailyData();
                break;
        }
    }


    @Override
    public void onGetDailyDataSuc(NewsBean newsBean) {

        mTv.setText(newsBean.toString());

    }

    @Override
    public void onGetDailyDataFailed(String msg) {
        Toast.makeText(this, msg, Toast.LENGTH_SHORT).show();
    }
}
```
#### MainPresenter
```java
public class MainPresenter extends BasePresenter<MainContact.View> implements MainContact.Presenter {

    public MainPresenter(MainContact.View view) {
        super(view);
    }

    @Override
    public void GetDailyData() {
        Observable<NewsBean> newsData = mDataManager.mDailyRepository.getNewsData();

        addSubscription(newsData, new Subscriber<NewsBean>() {
            @Override
            public void onCompleted() {

            }

            @Override
            public void onError(Throwable e) {
                mView.onGetDailyDataFailed(e.getMessage());
            }

            @Override
            public void onNext(NewsBean newsBean) {
                mView.onGetDailyDataSuc(newsBean);
            }
        });
    }
}
```

>主要就是让View和Presenter之间相互引用，然后可以互调方法，加上这种设计模式之后，代码的思路，流程会清晰很多，View主要负责给用户的操作进行
反馈，Presenter则负责处理具体的网络请求以及逻辑等。到这里整个框架就基本上差不多了。还有一些细节有待完善。

[源代码已经上传到github](https://github.com/frankandroid/FrameWorkSample);
