---
title: aboout_dagger2
date: 2017-01-22 10:38:08
updated: 2017-01-22 10:38:08
categories: [第三方框架使用] 
tags: [dagger2]

---

## dagger2 简介
> dagger2 是一个依赖注入框架，能够使项目更加的解耦。


## 如何添加到项目中

> 1 在project的build.gradle里面添加apt插件。
```java
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.2.2'

        classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}
```
> 2 在app的中的build.gradle文件中添加配置
```java
apply plugin: 'com.android.application'
apply plugin: 'com.neenbedankt.android-apt'//应用添加的插件。

android {
    compileSdkVersion 25
    buildToolsVersion "25.0.2"
    defaultConfig {
        applicationId "com.hhly.dagger2sample"
        minSdkVersion 15
        targetSdkVersion 25
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    androidTestCompile('com.android.support.test.espresso:espresso-core:2.2.2', {
        exclude group: 'com.android.support', module: 'support-annotations'
    })
    compile 'com.android.support:appcompat-v7:25.1.0'
    compile 'com.android.support:design:25.1.0'
    testCompile 'junit:junit:4.12'
    apt 'com.google.dagger:dagger-compiler:2.7'//加入dagger2的依赖。
    compile 'com.google.dagger:dagger:2.7'// 加入dagger2的依赖。
    provided 'javax.annotation:jsr250-api:1.0'//加入dagger2的依赖。
}
}
```
## 依赖注入简介
> 依赖注入：就是目标类（目标类指的是这个类的成员变量中有其他类的对象）中所依赖的其他的类的初始化过程，
不是通过手动编码的方式创建，而是通过技术手段可以把其他的类的已经初始化好的实例自动注入到目标类中。

```java
 class A{
       B b = new B(...);
       C c = new C();
       D d = new D(new E());
       F f = new F(.....);
 }
```

>上面的代码中class A就是指的目标类，对于class E来说,class D是class E的目标类，同时,class D又是class A的成员变量。

>那么通过依赖注入，我们就可以这样写。

```java
class A{
        @Inject
        B b;
   }

   class B{
       @Inject
       B(){
       }
   }
```

>通过对class A中的成员变量b，以及class B中的构造方法，添加@inject注解。我们就让他们之间有了一种无形的联系，那么怎么把这种
无形的联系变成有形的联系呢，这个时候我们就需要用到另外一个注解@Component

### Component简介

>Component也是一个注解类，一个类要想是Component，必须用Component注解来标注该类，并且该类是接口或抽象类
Component需要引用到目标类的实例，Component会查找目标类中用Inject注解标注的属性，
查找到相应的属性后会接着查找该属性对应的用Inject标注的构造函数（这时候就发生联系了），
剩下的工作就是初始化该属性的实例并把实例进行赋值。看代码：

> 1,建立component对象。

```java
@Component
public interface FirstComponent {

    //对scorllingActivity进行依赖注入。
    void injectScrollingActivity(ScrollingActivity activity);

}

```

> 2,在目标类中，对FirstComponent进行实例化，并进行依赖注入操作。
```java

    @Inject
    FirstStudent mFirstStudent;

    private FirstComponent mFirstComponent;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_scrolling);
        Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);

        //建立联系
        mFirstComponent = DaggerFirstComponent.builder().build();
        mFirstComponent.injectScrollingActivity(this);

        FloatingActionButton fab = (FloatingActionButton) findViewById(R.id.fab);
        fab.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Snackbar.make(view, "student age is" + mFirstStudent.age, Snackbar.LENGTH_LONG)
                        .setAction("Action", null).show();
            }
        });
    }
```
> 到此，通过component,进行依赖注入，不需要new对象，就能够获取到FirstStudent的实例，但是现在有这样一种情况，如果是我们自己的类，我们当然可以在我们的构造方法加上@Inject注解
但是我们在实际的开发中，经常会需要引用第三方的libs,这个时候我们需要实例化某个类，就没有办法在构造方法上加上@inject注解了。这个时候，我们就要用到@Module 和@Provides注解了

### Module和Provides
>Module 和 Provide主要就是为了解决第三方包的依赖注入问题，当然，一般我们自己写的类也会通过Module的形式进行注入，方便管理，一个可以同时提供几个Module,
Module里面提供相关类的实例。接下来看代码。

>首先，写一个类，加上@Module注解。然后在里面定义方法，提供相关的实例。这里我们以Gson为例
```java
//第三方的类
@Module
public class ThirdLibModule {


    @Provides
    Gson provideGson(){
        return new Gson();
    }

}

//自己定义的类
@Module
public class FirstModule {

    @Provides
    FirstStudent provideStudent(){
        return new FirstStudent(40);
    }
}

```

>接下来在component里面添加Module
```java
@Component(modules = {ThirdLibModule.class, FirstModule.class})
public interface ThirdLibComponent {

    void injectScrollingActivity(ScrollingActivity activity);

}
```

>最后在目标类中实例化Component,并注入目标类。

```java
package com.hhly.dagger2sample;

import android.os.Bundle;
import android.support.design.widget.FloatingActionButton;
import android.support.design.widget.Snackbar;
import android.support.v7.app.AppCompatActivity;
import android.support.v7.widget.Toolbar;
import android.view.Menu;
import android.view.MenuItem;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;

import com.google.gson.Gson;
import com.hhly.dagger2sample.bean.FirstStudent;
import com.hhly.dagger2sample.bean.GsonTestStudent;
import com.hhly.dagger2sample.di.component.DaggerThirdLibComponent;
import com.hhly.dagger2sample.di.component.FirstComponent;
import com.hhly.dagger2sample.di.component.ThirdLibComponent;
import com.hhly.dagger2sample.di.module.ThirdLibModule;

import javax.inject.Inject;

public class ScrollingActivity extends AppCompatActivity implements View.OnClickListener {

    private String mStudentJson = "{\n" +
            "    \"name\": \"Jack\",\n" +
            "    \"age\": 18,\n" +
            "    \"isBoy\": true\n" +
            "}";

    @Inject
    FirstStudent mFirstStudent;
    @Inject
    Gson mGson;

    private FirstComponent mFirstComponent;
    private ThirdLibComponent mThirdLibComponent;
    private TextView mShowContent;
    private GsonTestStudent mGsonTestStudent;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_scrolling);
        Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);
        setSupportActionBar(toolbar);

        //使用component直接建立联系
        /*mFirstComponent = DaggerFirstComponent.builder().build();
        mFirstComponent.injectScrollingActivity(this);*/

        //使用module方式提供实例
        mThirdLibComponent = DaggerThirdLibComponent.builder()
                .thirdLibModule(new ThirdLibModule())
                .build();

        mThirdLibComponent.injectScrollingActivity(this);

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
    public boolean onCreateOptionsMenu(Menu menu) {
        // Inflate the menu; this adds items to the action bar if it is present.
        getMenuInflater().inflate(R.menu.menu_scrolling, menu);
        return true;
    }

    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        // Handle action bar item clicks here. The action bar will
        // automatically handle clicks on the Home/Up button, so long
        // as you specify a parent activity in AndroidManifest.xml.
        int id = item.getItemId();

        //noinspection SimplifiableIfStatement
        if (id == R.id.action_settings) {
            return true;
        }
        return super.onOptionsItemSelected(item);
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
                mShowContent.setText(mGsonTestStudent.toString()+"///"+mFirstStudent.toString());
                break;
        }
    }
}

```

#### 注意:由于之前是通过注解的方式进行的依赖，所以FirstComponent中还保留着以下代码，会出bug,这个bug要注意。所以把代码注释掉。

```java
@Component()
public interface FirstComponent {

    /**
     * 注意一个类不能同时被两个component注入，因为这些东西是在编译的时候生成的，那么在编译的时候，你一个类同时
     * 被两个component注入，那目标类会不知道去哪个component里面找，于是就去第一个找，结果找不到，于是就会
     * 报错。
     */
   /* void injectScrollingActivity(ScrollingActivity activity);*/
}
```

#### 第一篇先写这些，第二篇会接着介绍其他的注解以及其含义。

