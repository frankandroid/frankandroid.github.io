---
title: RxJava一些常用操作符的总结汇总
date: 2017-10-06 10:53:14
updated: 2017-10-06 10:53:14
categories: [第三方框架使用] 
tags: [RxJava]
---

>RxJava现在非常的火，我们的项目中也采用了Rxjava+Dagger2+Mvp的架构，但是一直没有时间对这一块常用的操作符进行很好的总结。今天抓紧时间，把以前看过的博客重新梳理了一下，这里重点推荐两个博客，一个是[大头鬼翻译的系列博客](http://blog.csdn.net/lzyzsd/article/details/41833541/),
另一个是[扔物线的给Android开发者的RxJava 详解](http://gank.io/post/560e15be2dca930e00da1083)，看完这两个感觉就会清晰很多。一些常用的操作符也会很清晰。说白了，RaJava主要做了两件事，一是一行代码进行线程切换，二是对数据源进行转换，转换成我们需要的数据。


# 创建观察者(Observer)

```java
//第一种
Observer<String> observer = new Observer<String>() {
    @Override
    public void onNext(String s) {
        Log.d(tag, "Item: " + s);
    }

    @Override
    public void onCompleted() {
        Log.d(tag, "Completed!");
    }

    @Override
    public void onError(Throwable e) {
        Log.d(tag, "Error!");
    }
};

//第二种
Subscriber<String> subscriber = new Subscriber<String>() {
    @Override
    public void onNext(String s) {
        Log.d(tag, "Item: " + s);
    }

    @Override
    public void onCompleted() {
        Log.d(tag, "Completed!");
    }

    @Override
    public void onError(Throwable e) {
        Log.d(tag, "Error!");
    }
};

```
>建议用第二种，在RxJava的subscribe过程中，Observer 也总是会先被转换成一个 Subscriber 再使用。

# 创建被观察者(Observable)

```java
//第一种，用的比较少
Observable observable = Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("Hello");
        subscriber.onNext("Hi");
        subscriber.onNext("Aloha");
        subscriber.onCompleted();
    }
});

//第二种，用的比较多，就是依次发送，just操作符。
Observable observable = Observable.just("Hello", "Hi", "Aloha");

//第三种，也用的比较多，依次发送一个数组或者集合里面的元素，from操作符。

String[] words = {"Hello", "Hi", "Aloha"};
Observable observable = Observable.from(words);

```

# 订阅

> 由于我们会经常性的对数据源进行转换，所以是被观察者Observable在前，观察者Observer在后.

```java

observable.subscribe(subscriber);
```

# Action1 和 Action0

> 有的时候你也可以不传入Subscriber,直接传入Action,

```java
//有参有返回值
Action1<String> onNextAction = new Action1<String>() {
    // onNext()
    @Override
    public void call(String s) {
        Log.d(tag, s);
    }
};
Action1<Throwable> onErrorAction = new Action1<Throwable>() {
    // onError()
    @Override
    public void call(Throwable throwable) {
        // Error handling
    }
};

//无参无返回值
Action0 onCompletedAction = new Action0() {
    // onCompleted()
    @Override
    public void call() {
        Log.d(tag, "completed");
    }
};

// 自动创建 Subscriber ，并使用 onNextAction 来定义 onNext()
observable.subscribe(onNextAction);
// 自动创建 Subscriber ，并使用 onNextAction 和 onErrorAction 来定义 onNext() 和 onError()
observable.subscribe(onNextAction, onErrorAction);
// 自动创建 Subscriber ，并使用 onNextAction、 onErrorAction 和 onCompletedAction 来定义 onNext()、 onError() 和 onCompleted()
observable.subscribe(onNextAction, onErrorAction, onCompletedAction);

```

# 线程切换，Scheduler （很重要）
>在不指定线程的情况下， RxJava 遵循的是线程不变的原则，即：在哪个线程调用 subscribe()，就在哪个线程生产事件；在哪个线程生产事件，就在哪个线程消费事件。如果需要切换线程，就需要用到 Scheduler （调度器）。

## 常用的几个Scheduler
* Schedulers.immediate(): 直接在当前线程运行，相当于不指定线程。这是默认的 Scheduler
* Schedulers.newThread(): 总是启用新线程，并在新线程执行操作。
* Schedulers.io(): **(非常的常用)**I/O 操作（读写文件、读写数据库、网络信息交互等）所使用的 Scheduler。行为模式和 newThread() 差不多，区别在于 io() 的内部实现是是用一个无数量上限的线程池，可以重用空闲的线程，因此多数情况下 io() 比 newThread() 更有效率。不要把计算工作放在 io() 中，可以避免创建不必要的线程。
* Schedulers.computation(): 计算所使用的 Scheduler。这个计算指的是 CPU 密集型计算，即不会被 I/O 等操作限制性能的操作，例如图形的计算。这个 Scheduler 使用的固定的线程池，大小为 CPU 核数。不要把 I/O 操作放在 computation() 中，否则 I/O 操作的等待时间会浪费 CPU。
* AndroidSchedulers.mainThread(): **(非常的常用)**它指定的操作将在 Android 主线程运行。


##  subscribeOn() 和 observeOn(),经常容易混淆。

 * subscribeOn(): 指定 subscribe() 所发生的线程，即 Observable.OnSubscribe 被激活时所处的线程。或者叫做事件产生的线程。 
 * observeOn(): 指定 Subscriber 所运行在的线程。或者叫做事件消费的线程。

```java
//以下代码在安卓开发中非常的常见
Observable.just(1, 2, 3, 4)
    .subscribeOn(Schedulers.io()) // 指定 subscribe() 发生在 IO 线程
    .observeOn(AndroidSchedulers.mainThread()) // 指定 Subscriber 的回调发生在主线程
    .subscribe(new Action1<Integer>() {
        @Override
        public void call(Integer number) {
            Log.d(tag, "number:" + number);
        }
    });
```

## 需要注意的几个细节
1. observeOn() 指定的是它之后的操作所在的线程,可以多次调用，调用的顺序有影响。
2. subscribeOn() 的位置放在哪里都可以，但是最先调用的一次有效果。（说只能调用一次也不太准确，因为在doOnSubscriber()的时候会调用离这个方法最近的subscribeOn()所在的线程。后面会有相关的描述。）




# 变换

## map操作符
>用于将一种数据转换成另一种数据，例如传入一个String类型的字符串，返回一个Bitmap对象，

```java
Observable.just("images/logo.png") // 输入类型 String
    .map(new Func1<String, Bitmap>() {
        @Override
        public Bitmap call(String filePath) { // 参数类型 String
            return getBitmapFromPath(filePath); // 返回类型 Bitmap
        }
    })
    .subscribe(new Action1<Bitmap>() {
        @Override
        public void call(Bitmap bitmap) { // 参数类型 Bitmap
            showBitmap(bitmap);
        }
    })

```

## Func
>Func1 和 Action1 非常相似，也是 RxJava 的一个接口，用于包装含有一个参数的方法。 Func1 和 Action 的区别在于， Func1 包装的是有返回值的方法。


## flatmap

>flatmap 返回一个Observable对象，但是Subsciber接收到的是Observable对象的泛型，具体点说呢就是flatMap() 和 map() 有一个相同点：它也是把传入的参数转化之后返回另一个对象。但需要注意，和 map() 不同的是， flatMap() 中返回的是个 Observable 对象，并且这个 Observable 对象并不是被直接发送到了 Subscriber 的回调方法中。 flatMap() 的原理是这样的：
1. 使用传入的事件对象创建一个 Observable 对象；
2. 并不发送这个 Observable, 而是将它激活，于是它开始发送事件；
3. 每一个创建出来的 Observable 发送的事件，都被汇入同一个 Observable ，而这个 Observable 负责将这些事件统一交给 Subscriber 的回调方法。

这三个步骤，把事件拆成了两级，通过一组新创建的 Observable 将初始的对象『铺平』之后通过统一路径分发了下去。而这个『铺平』就是 flatMap() 所谓的 flat。

首先假设这么一种需求：有一个数据结构『学生』，现在需要打印出一组学生的名字。实现方式很简单：

```java
Student[] students = ...;
Subscriber<String> subscriber = new Subscriber<String>() {
    @Override
    public void onNext(String name) {
        Log.d(tag, name);
    }
    ...
};
Observable.from(students)
    .map(new Func1<Student, String>() {
        @Override
        public String call(Student student) {
            return student.getName();
        }
    })
    .subscribe(subscriber);

```

那么再假设：如果要打印出每个学生所需要修的所有课程的名称呢？（需求的区别在于，每个学生只有一个名字，但却有多个课程。）首先可以这样实现：
```java
Student[] students = ...;
Subscriber<Student> subscriber = new Subscriber<Student>() {
    @Override
    public void onNext(Student student) {
        List<Course> courses = student.getCourses();
        for (int i = 0; i < courses.size(); i++) {
            Course course = courses.get(i);
            Log.d(tag, course.getName());
        }
    }
    ...
};
Observable.from(students)
    .subscribe(subscriber);
```

那么如果我不想在 Subscriber 中使用 for 循环，而是希望 Subscriber 中直接传入单个的 Course 对象呢（这对于代码复用很重要）？用 map() 显然是不行的，因为 map() 是一对一的转化，而我现在的要求是一对多的转化。那怎么才能把一个 Student 转化成多个 Course 呢？
这个时候，就需要用 flatMap() 了：

```java
//这么做优雅多了。
Student[] students = ...;
Subscriber<Course> subscriber = new Subscriber<Course>() {
    @Override
    public void onNext(Course course) {
        Log.d(tag, course.getName());
    }
    ...
};
Observable.from(students)
    .flatMap(new Func1<Student, Observable<Course>>() {
        @Override
        public Observable<Course> call(Student student) {
            return Observable.from(student.getCourses());
        }
    })
    .subscribe(subscriber);
```

>一个小细节就是Func的泛型，Func<传入的类型，返回值的类型>。


## 转换的原理，其实就是把扔物线的话整理了一下，让自己更好明白。

订阅的原理

```java
// 注意：这不是 subscribe() 的源码，而是将源码中与性能、兼容性、扩展性有关的代码剔除后的核心代码。
// 如果需要看源码，可以去 RxJava 的 GitHub 仓库下载。
public Subscription subscribe(Subscriber subscriber) {
    subscriber.onStart();
    onSubscribe.call(subscriber);//调用Observable的call方法。把Subscriber传入进去。
    return subscriber;
}

Observable observable = Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("Hello");
        subscriber.onNext("Hi");
        subscriber.onNext("Aloha");
        subscriber.onCompleted();
    }
});

```

那么转换的原理和订阅的原理十分的类似，就是需要在Observable.onSubscribe的call方法中传入经过转换后的Subscriber。
```java
// 注意：这不是 lift() 的源码，而是将源码中与性能、兼容性、扩展性有关的代码剔除后的核心代码。
// 如果需要看源码，可以去 RxJava 的 GitHub 仓库下载。
public <R> Observable<R> lift(Operator<? extends R, ? super T> operator) {
    return Observable.create(new OnSubscribe<R>() {
        @Override
        public void call(Subscriber subscriber) {//
            Subscriber newSubscriber = operator.call(subscriber);//注意这个生成的新的订阅者，传入旧的订阅者，生成新的订阅者。
            newSubscriber.onStart();
            onSubscribe.call(newSubscriber);//最终call()方法中传入的是经过转换后的新的Subscriber。这样最终Observable被订阅的时候，就是调用的新的Subscriber的方法。
        }
    });
}
```
精简掉细节的话，也可以这么说：在 Observable 执行了 lift(Operator) 方法之后，会返回一个新的 Observable，这个新的 Observable 会像一个代理一样，负责接收原始的 Observable 发出的事件，并在处理后发送给 Subscriber。


# Compose 对 Observable 整体的变换

>Compose操作符适用于当需要对一大批Observable进行整体变换的场合。

```java
public class LiftAllTransformer implements Observable.Transformer<Integer, String> {
    @Override
    public Observable<String> call(Observable<Integer> observable) {
        return observable
            .lift1()
            .lift2()
            .lift3()
            .lift4();
    }
}
...
Transformer liftAll = new LiftAllTransformer();
observable1.compose(liftAll).subscribe(subscriber1);
observable2.compose(liftAll).subscribe(subscriber2);
observable3.compose(liftAll).subscribe(subscriber3);
observable4.compose(liftAll).subscribe(subscriber4);
```

# doOnSubscribe()
>它和 Subscriber.onStart() 同样是在 subscribe() 调用后而且在事件发送前执行，
但区别在于它可以指定线程。默认情况下， doOnSubscribe() 执行在 subscribe() 发生的线程；而如果在 doOnSubscribe() 之后有 subscribeOn() 的话，它将执行在离它最近的 subscribeOn() 所指定的线程

```java
Observable.create(onSubscribe)
    .subscribeOn(Schedulers.io())
    .doOnSubscribe(new Action0() {
        @Override
        public void call() {
            progressBar.setVisibility(View.VISIBLE); // 需要在主线程执行
        }
    })
    .subscribeOn(AndroidSchedulers.mainThread()) // 指定主线程
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(subscriber);
```


>就是这么多了。