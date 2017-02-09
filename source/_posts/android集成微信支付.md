---
title: android集成微信支付
date: 2017-02-09 15:11:04
updated: 2017-02-09 15:11:04
categories: [支付]
tags: [支付]
---

# android集成微信支付

1. [先看微信的官方文档](https://pay.weixin.qq.com/wiki/doc/api/app/app.php?chapter=8_5#);

2. 根据应用的包名生成应用的签名，注意需要是正式包，然后配置到后台
<img src="https://i.niupic.com/images/2017/02/09/ZnwO60.png" />
<img src="https://i.niupic.com/images/2017/02/09/MOLd6E.png" />

3. 把微信提供的jar包放入到libs目录下，在Manifest文件中配置好WXPayEntryActivity（看第四步）。

<img src="https://i.niupic.com/images/2017/02/09/UHOAx0.png" />


```java
 <!-- 微信支付 -->
        <activity
            android:name=".wxapi.WXPayEntryActivity"
            android:exported="true"
            android:launchMode="singleTop" />

```


4. 必须在项目包名的根目录下建一个名称为"wxapi"的文件夹。然后放入WXPayEntryActivity类。这个细节很重要。
<img src="https://i.niupic.com/images/2017/02/09/sIJEKh.png" />


5. 去后台请求数据

```java
@Override
    public void goToWePay(String orderId) {

        Observable<BaseBean<WePayResBean>> wePayObservable = getDataStore().goToWePay(orderId, Constant.Pay.WE_PAY);

        wePayObservable.compose(this.<BaseBean<WePayResBean>>bindToLifecycle())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Subscriber<BaseBean<WePayResBean>>() {
                    @Override
                    public void onCompleted() {

                    }

                    @Override
                    public void onError(Throwable e) {

                    }

                    @Override
                    public void onNext(BaseBean<WePayResBean> wePayResBeanBaseBean) {
                        if (wePayResBeanBaseBean.code==Constant.SUC){
                            mView.onGoToWePaySuc(wePayResBeanBaseBean.data);
                        }else{
                            mView.onGoToWePayFailed(wePayResBeanBaseBean.msg);
                        }
                    }
                });

    }
```
6. 获取数据的bean

```java
public class WePayResBean {
    
    public String appid;
    public String noncestr;
    public String packageValue;
    public String partnerid;
    public String prepayid;
    public String sign;
    public String timestamp;

    
}

```

7. 必须在项目包名的根目录下建一个名称为"wxapi"的文件夹。然后放入WXPayEntryActivity类。这个细节很重要。
<img src="https://i.niupic.com/images/2017/02/09/sIJEKh.png" />


8. 获取数据成功后。调用微信的API。
```java
 @Override
    public void onGoToWePaySuc(WePayResBean wePayResBean) {

        final IWXAPI msgApi = WXAPIFactory.createWXAPI(mActivity, null);
        msgApi.registerApp(wePayResBean.getAppid());

        PayReq request = new PayReq();
        request.appId = wePayResBean.getAppid();
        request.partnerId = wePayResBean.getPartnerid();
        request.prepayId = wePayResBean.getPrepayid();
        request.packageValue = "Sign=WXPay";
        request.nonceStr = wePayResBean.getNoncestr();
        request.timeStamp = wePayResBean.getTimestamp();
        request.sign = wePayResBean.getSign();

        msgApi.sendReq(request);
    }
```

9. 支付返回的结果会发送到WXPayEntryActivity类中。

```java

//这个类是微信提供的类。
public class WXPayEntryActivity extends BaseActivity implements IWXAPIEventHandler, View.OnClickListener {

    private static final String TAG = "MicroMsg.SDKSample.WXPayEntryActivity";
    @BindView(R.id.toolbar)
    SimpleToolbar mToolbar;
    @BindView(R.id.pay_suc_iv)
    ImageView     mPaySucIv;
    @BindView(R.id.pay_failed_iv)
    ImageView     mPayFailedIv;
    @BindView(R.id.thank_tv)
    TextView      mThankTv;
    @BindView(R.id.go_to_tv)
    TextView      mGoToTv;
    @BindView(R.id.pay_order_suc_ll)
    LinearLayout  mPayOrderSucLl;
    @BindView(R.id.pay_result_tv)
    TextView      mPayResultTv;

    private IWXAPI api;
    private boolean isPaySuc;
    private Handler mHandler;
    private boolean mIsFromRed;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_pay_success);
        ButterKnife.bind(this);

        //初始化
        mHandler = new Handler();
        api = WXAPIFactory.createWXAPI(this, Constant.APP_ID);
        api.handleIntent(getIntent(), this);

        mToolbar.setOnNavigationClickListener(this);
        mGoToTv.setOnClickListener(this);
        
    }

    @Override
    protected int onGetLayoutId() {
        return R.layout.activity_pay_success;
    }

    //必须复写
    @Override
    protected void onNewIntent(Intent intent) {
        super.onNewIntent(intent);
        setIntent(intent);
        api.handleIntent(intent, this);
    }

    @Override
    public void onReq(BaseReq req) {
    }

    /**支付成功与否的回掉*/
    @Override
    public void onResp(BaseResp resp) {

        if (resp.getType() == ConstantsAPI.COMMAND_PAY_BY_WX) {
            if (resp.errCode == 0) {
                setSuc();
            } else {
                setFailed();
            }
        }
    }


    /**
     * 支付成功后显示的界面
     */
    public void setSuc() {
        isPaySuc = true;
        mPaySucIv.setVisibility(View.VISIBLE);
        mPayFailedIv.setVisibility(View.GONE);
        mThankTv.setVisibility(View.VISIBLE);
        mPayOrderSucLl.setVisibility(View.VISIBLE);

        mIsFromRed = PreferencesUtils.getBoolean(this, Constant.Order.IS_FORM_RED);
        if (mIsFromRed){
            mHandler.postDelayed(new Runnable() {
                @Override
                public void run() {
                   goChatActivity();
                }
            },500);
        }
    }

    /**
     * 支付失败后显示的界面
     */

    public void setFailed() {
        isPaySuc = false;
        mPayResultTv.setText(getString(R.string.pay_failed));
        mPayFailedIv.setVisibility(View.VISIBLE);
        mPaySucIv.setVisibility(View.GONE);
        mThankTv.setVisibility(View.GONE);
        mPayOrderSucLl.setVisibility(View.GONE);
    }

```

>需要注意的有3点，第一点是：必须要在项目包名的根目录下，项目包名的根目录下建一个名称为"wxapi"的文件夹。然后放入WXPayEntryActivity类。
第二个是支付结果的回掉在WXPayEntryActivity中。第三点就是生成的配置到后台的签名必须是正式的签名。


[欢迎下载我们的app:律正,在您需要帮助的时候给你提供法律咨询](http://www.962.net/azgame/142520.html);




