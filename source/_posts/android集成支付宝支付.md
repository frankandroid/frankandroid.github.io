---
title: android集成支付宝支付
date: 2017-02-09 10:21:13
updated: 2017-02-09 10:21:13
categories: [支付]
tags: [支付]
---

# android集成支付宝支付

1. [首先当然是先看支付宝的官方文档](https://doc.open.alipay.com/docs/doc.htm?spm=a219a.7629140.0.0.s9V7dl&treeId=59&articleId=103563&docType=1);

2. 在项目的libs目录下放入支付宝的jar包。
<img src="https://i.niupic.com/images/2017/02/09/l5WuJG.png" />

3. 修改Manifest

```java
<!-- 支付宝相关的activity -->
        <activity
            android:name="com.alipay.sdk.app.H5PayActivity"
            android:configChanges="orientation|keyboardHidden|navigation"
            android:exported="false"
            android:screenOrientation="behind" />
        <activity
            android:name="com.alipay.sdk.auth.AuthActivity"
            android:configChanges="orientation|keyboardHidden|navigation"
            android:exported="false"
            android:screenOrientation="behind" />
```

    加入权限
```java
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.READ_PHONE_STATE" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
```


4. 去后台请求数据

```java
@Override
    public void goToAliPay(String orderId) {
        Observable<BaseBean<String>> aliPayObservable = getDataStore().goToAliPay(orderId, Constant.Pay.ALI_PAY);
        aliPayObservable.compose(this.<BaseBean<String>>bindToLifecycle())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Subscriber<BaseBean<String>>() {
                    @Override
                    public void onCompleted() {

                    }
                    @Override
                    public void onError(Throwable e) {
                        e.printStackTrace();
                    }
                    @Override
                    public void onNext(BaseBean<String> stringBaseBean) {
                         if (stringBaseBean.code==Constant.SUC){
                             mView.onGoToAliPaySuc(stringBaseBean.data);
                         }else{
                             mView.onGoToAliPayFailed();
                         }
                    }
                });
    }

```
5. 拿到后台返回的String字符串。

```java
    @Override
    public void onGoToAliPaySuc(final String aliMsg) {

        Runnable payRunnable = new Runnable() {
            @Override
            public void run() {
                PayTask alipay = new PayTask(mActivity);
                String result = alipay.pay(aliMsg, true);
                Message msg = new Message();
                msg.what = SDK_PAY_FLAG;
                msg.obj = result;
                mHandler.sendMessage(msg);
            }
        };
        // 必须异步调用
        Thread payThread = new Thread(payRunnable);
        payThread.start();

    }
```

6. 复制PayResult类到项目中。后面会用到。
```java

public class PayResult {

	private String resultStatus;
	private String result;
	private String memo;

	public PayResult(String rawResult) {

		if (TextUtils.isEmpty(rawResult))
			return;

		String[] resultParams = rawResult.split(";");
		for (String resultParam : resultParams) {
			if (resultParam.startsWith("resultStatus")) {
				resultStatus = gatValue(resultParam, "resultStatus");
			}
			if (resultParam.startsWith("result")) {
				result = gatValue(resultParam, "result");
			}
			if (resultParam.startsWith("memo")) {
				memo = gatValue(resultParam, "memo");
			}
		}
	}

	@Override
	public String toString() {
		return "resultStatus={" + resultStatus + "};memo={" + memo
				+ "};result={" + result + "}";
	}

	private String gatValue(String content, String key) {
		String prefix = key + "={";
		return content.substring(content.indexOf(prefix) + prefix.length(),
				content.lastIndexOf("}"));
	}

	/**
	 * @return the resultStatus
	 */
	public String getResultStatus() {
		return resultStatus;
	}

	/**
	 * @return the memo
	 */
	public String getMemo() {
		return memo;
	}

	/**
	 * @return the result
	 */
	public String getResult() {
		return result;
	}
}

```

7. 在Handler里面接受消息。

```java

private Handler mHandler = new Handler() {

        public void handleMessage(Message msg) {
            switch (msg.what) {

                case SDK_PAY_FLAG: {
                    PayResult payResult = new PayResult((String) msg.obj);
                    /**
                     * 同步返回的结果必须放置到服务端进行验证，建议商户依赖异步通知
                     */
                    String resultInfo = payResult.getResult();// 同步返回需要验证的信息

                    String resultStatus = payResult.getResultStatus();
                    // 判断resultStatus 为“9000”则代表支付成功，具体状态码代表含义可参考接口文档
                    if (TextUtils.equals(resultStatus, "9000")) {
                        Toast.makeText(mActivity, "支付成功", Toast.LENGTH_SHORT).show();
                        
                        //跳转到展示页面
                        Intent intent = new Intent(mActivity, WXPayEntryActivity.class);
                        intent.putExtra(Constant.Pay.ALI_PAY_STATUES, true);
                        intent.putExtra(Constant.Pay.IS_FROM_ALI, true);
                        startActivity(intent);
                    } else {
                        // 判断resultStatus 为非"9000"则代表可能支付失败
                        // "8000"代表支付结果因为支付渠道原因或者系统原因还在等待支付结果确认，最终交易是否成功以服务端异步通知为准（小概率状态）
                        if (TextUtils.equals(resultStatus, "8000")) {
                            Toast.makeText(mActivity, "支付结果确认中", Toast.LENGTH_SHORT).show();
                        } else {
                            // 其他值就可以判断为支付失败，包括用户主动取消支付，或者系统返回的错误
                            Toast.makeText(mActivity, "支付失败", Toast.LENGTH_SHORT).show();
                        }
                        //跳转到展示页面
                        Intent intent = new Intent(mActivity, WXPayEntryActivity.class);
                        intent.putExtra(Constant.Pay.IS_FROM_ALI, true);
                        intent.putExtra(Constant.Pay.ALI_PAY_STATUES, false);
                        startActivity(intent);
                    }
                    break;
                }
            }
        }
    };

```

>如果发现掉起了支付宝客户端，但是支付失败，那么多半是后台返回的数据有问题（之前项目中遇到过）。
<img src="https://i.niupic.com/images/2017/02/09/mI0hZX.png" />

>整个流程大概就是这样，主要是写移动端集成的流程，方便以后复用。



