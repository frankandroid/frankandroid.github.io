---
title: GreenDao3.0的使用
date: 2016-12-09 20:25:00
updated: 2016-12-09 20:25:00
categories: [第三方框架使用]
tags: [GreenDao3.0]
---

>在APP开发中，经常会需要使用到数据库，用SqliteOpenHelper效率不是很高，也更麻烦，所以用GreenDao会高效一些。

## 1. 首先在项目中导入GreenDao的库。

在project的build.gradle中，添加插件。

```java
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.2.2'
        classpath 'org.greenrobot:greendao-gradle-plugin:3.2.1'
        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}

```

在app的buile.gradle中添加依赖，并加上greeddao的配置。

```java
apply plugin: 'com.android.application'
apply plugin: 'org.greenrobot.greendao'

android {
    compileSdkVersion 25
    buildToolsVersion "25.0.2"
    defaultConfig {
        applicationId "com.hhly.greendaodemo"
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

    greendao {
        schemaVersion 1
        daoPackage 'com.hhly.greendaodemo.gen'
        targetGenDir 'src/main/java'
    }

}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    androidTestCompile('com.android.support.test.espresso:espresso-core:2.2.2', {
        exclude group: 'com.android.support', module: 'support-annotations'
    })
    compile 'com.android.support:appcompat-v7:25.1.0'
    testCompile 'junit:junit:4.12'
    compile 'org.greenrobot:greendao:3.2.0'
}
```

>schemaVersion： 表示的数据库schema的版本号：在升级数据库或者数据迁移的时候需要改变这个值，只要每次都+1就行；
 daoPackage： dao的包名，包名默认是entity所在的包；
 targetGenDir : 自动生成的数据库文件的目录。

如果上面不指定目录的话，数据库文件就会生成到build/generated/source/greendao下面，还需要拷贝到自己的目录中，所以这里还是指定生成目录为好。


## 2. 建实体bean.

```java
@Entity
public class User {

    @Id
    private Long id;
    @Property(nameInDb = "USERNAME")
    private String username;
    @Property(nameInDb = "NICKNAME")
    private String nickname;

}
```

>一些常用的注解

<img src="https://i.niupic.com/images/2017/02/10/UZlYU6.png" />

OK，写完这些之后将项目进行编译，编译成功之后系统会帮助我们生成相应的构造方法和get/set方法，并且还会在我们的包下生成DaoMaster和DaoSession。

build之后，项目下会自动生成一些类，User类也会自动生成一些东西

<img src="https://i.niupic.com/images/2017/02/10/1UuQ5W.png" />

```java
@Entity
public class User {

    @Id
    private Long id;
    @Property(nameInDb = "USERNAME")
    private String username;
    @Property(nameInDb = "AGE")
    private String age;
    @Generated(hash = 1942375784)
    public User(Long id, String username, String age) {
        this.id = id;
        this.username = username;
        this.age = age;
    }
    @Generated(hash = 586692638)
    public User() {
    }
    public Long getId() {
        return this.id;
    }
    public void setId(Long id) {
        this.id = id;
    }
    public String getUsername() {
        return this.username;
    }
    public void setUsername(String username) {
        this.username = username;
    }
    public String getAge() {
        return this.age;
    }
    public void setAge(String age) {
        this.age = age;
    }
}
```

## 3.写一个对数据库进行操作的Util.在构造方法中对获取DaoSession,然后获取到UserDao。

```java
public class DBUtil {

    private DaoSession mDaoSession;
    private UserDao mUserDao;
    public DBUtil(Context context) {

        DaoMaster.DevOpenHelper devOpenHelper = new DaoMaster.DevOpenHelper(context, "user.db",
                null);
        DaoMaster daoMaster = new DaoMaster(devOpenHelper.getWritableDb());
        mDaoSession = daoMaster.newSession();
        mUserDao = mDaoSession.getUserDao();
    }

    /**
     * 增
     */
    public void insertUser(User user) {
        mUserDao.insert(user);
    }
    /**
     * 删一个
     */
    public void deleteOne(int id) {
        User user = mUserDao.queryBuilder().where(UserDao.Properties.Id.eq(id)).build().unique();
        mUserDao.delete(user);
    }
    /**
     * 删全部
     */
    public void deleteAll() {
        mUserDao.deleteAll();
    }
    /**
     * @param id 需要更新的age值。
     * 改
     */
    public void modify(int id) {

        User user = mUserDao.queryBuilder().where(UserDao.Properties.Id.eq(id)).build().unique();
        user.setUsername("张三被修改成了李四");
        mUserDao.update(user);
    }
    /**
     * 查一个
     */
    public User queryOne(int id) {

        User user = mUserDao.queryBuilder().where(UserDao.Properties.Id.eq(id)).build().unique();
        return user;
    }

    /**
     * 查询一批
     * @param startId 起始id
     * @param endId 结束id
     *
     */
    public List<User> querySome(int startId,int endId) {

        List<User> users = mUserDao.queryBuilder().where(UserDao.Properties.Id.between(startId,
                endId)).build().list();
        return users;
    }
    
    /**
     * 查全部
     */
    public List<User> queryAll() {
        return mUserDao.loadAll();
    }
}

```

调试数据库的时候，建议使用steto进行调试。

<img src="https://i.niupic.com/images/2017/02/10/c0i1wv.png" />


[源代码已经上传到github](https://github.com/frankandroid/GreenDaoDemo.git);





