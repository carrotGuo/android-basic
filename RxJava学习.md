> RxJava的好处在于，随着程序逻辑变得越来越复杂，它依然能够保持简洁。

# RxJava的设计模式
RxJava采用的是设计者模式，被观察者是Observable，观察者是Observer，被观察者会在一个序列中发送一系列消息，而Observer(观察者)会订阅（注册）被观察者(Observable)，并且观察者会对来自被观察者的一系列消息进行处理。一般来说，Observable会通过调用onNext()、onError()、onCompleted()通知观察者。

# 五种观察者模式
![](https://note.youdao.com/yws/public/resource/4fbcae57ed9c225f22300c9968c9ee8b/xmlnote/418C52BBD83F41DEA560DA96DFB446C8/15676)


# RxJava的四个基本概念
- #### Observable(被观察者)      
被观察者，决定什么时候触发事件和触发什么事件；
```
Observable observable = Observable.create(new Observable.OnSubscribe<String>() {
    @Override
    public void call(Subscriber<? super String> subscriber) {
        //一系列订阅后触发的事件
        subscriber.onNext("Hello");
        subscriber.onNext("Hi");
        subscriber.onNext("Aloha");
        subscriber.onCompleted();
    }
});
```

- #### Observer(观察者)      
观察者，决定被触发时有怎样的行为；
```
Observer<String> observer = new Observer<String>() {
    @Override
    public void onNext(String s) {
        //收到一个事件
        Log.d(tag, "Item: " + s);
    }

    @Override
    public void onCompleted() {
        //事件序列已经发送完成，接下来没有事件
        Log.d(tag, "Completed!");
    }

    @Override
    public void onError(Throwable e) {
        //某个事件发生错误，接下来不会调用onCompleted()
        Log.d(tag, "Error!");
    }
};
```

- #### subscribe()(订阅)     
将Observable和Observe连接起来
```
observable.subscribe(observer);
```
而在订阅的内部中，主要是调用了观察者的onStart()方法，接下来再调用observable的call方法，call中主要是对事件进行发送，这样就能达到被观察者发送通知事件给观察者的效果，并且可看出，必须要订阅了才会触发call，也就是订阅后被观察者才会发送消息；

- #### 事件

# RxJava几个重要方法
- onNext()      
当有一条消息发出时调用，序列中每发出一个事件就调用一次onNext();

- onError()     
当发送事件时发生了某些错误，此后Observable的序列中不会再发送事件，此时队列自动终止，且不会调用onCompleted()；

- onCompleted()     
当序列中的事件发送完毕时，会触发onCompleted()；

- onStart()     
在事件发送前触发，可以用于做一些准备工作，例如列表数据清零；（Subscriber才有的方法）

- unSubscribe()         
用于让观察者解除与被观察者的订阅，防止内存泄露，在onPause()或onStop()中调用；（Subscriber才有的方法）

# Observable/Observer设计模式
- #### 使用三步骤
1. 创建Observable：被观察者;
2. 创建Observer：观察者;
3. 使用subscribe()进行订阅；
```
Observable.just("hello world")
        .subscribe(new Consumer<String>() {
            @Override
            public void accept(String s) throws Exception {
                Log.i("success accept", s);
            }
        }, new Consumer<Throwable>() {
            @Override
            public void accept(Throwable throwable) throws Exception {
                Log.e("error", throwable.getMessage());
            }
        }, new Action() {
            @Override
            public void run() throws Exception {
                Log.e("complete","onComplete");
            }
        });
```
just() 是Observeble，将事件提交到流中，作为上游；     
Consumer、Action等参数是Observer,是下游，可以在某个时刻响应Observable，在不同线程中执行任务，非阻塞；      
subscribe() 作为连接线，将observable和observer进行连接，使上下游实现链式调用。     
- ##### Consumer和Action的区别
Consumer:单一参数类型；     
Action：无参数类型；

- #### RxJava2使用改进
> 由于RxJava2不支持订阅Subscriber，因此可以使用Observer作为观察者，改进如下：
```
Observable.just("hello world")
                .subscribe(new Observer<String>() {
                    @Override
                    public void onSubscribe(Disposable d) {
                        //事件发送时就会调用
                        Log.i("first do","subscribe");
                    }

                    @Override
                    public void onNext(String s) {
                        //收到一个消息时调用
                        Log.i("success accept", s);
                    }

                    @Override
                    public void onError(Throwable e) {
                        Log.e("error", e.getMessage());
                    }

                    @Override
                    public void onComplete() {
                        Log.e("complete", "onComplete");
                    }
                });
        //output:subscribe  hello world  onComplete
```
必须调用subscribe()后，被观察者才会发送数据；

#### Observable分类
- ##### HotObservable
无论有误观察者订阅，事件始终会发生；一对多关系，可以与多个订阅者共享数据；
- ##### ColdObservable
只有观察者订阅，才会执行发射数据流的代码；一对一关系。有多个订阅者时，消息是重新完整发送的。

#### 不完整定义的回调
它会根据回调创建Subcriber，Action1、Action0都是RxJava的接口，后面数字表示参数个数；(RxJava1)
```
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

# 线程切换
#### 两次切换
一般subscribeOn()指定Observable的OnSubscribe执行的线程，即触发事件所在线程，OnSubscribe中保存一个订阅者表，用于调用订阅者的几个主要方法；observeOn()方法是observer收到事件并做相应处理的所在线程；
```
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

#### 多次切换
一般subscribeOn()指定Observable的OnSubscribe执行的线程，即触发事件所在线程。observeOn()方法一般切换其后操作的过程，每个过程的线程对应其之前且最近的那个oserveOn()所指定的线程。而subscribeOn()只有第一个生效，除非有doOnSubscribe()。

#### doOnSubscribe()
由于observable触发事件是异步的，一般不在主线程，而subscriber.onStart()方法是在订阅时触发，此时事件还未触发，适合用于处理一些事件处理前的操作，但非常遗憾的是这个方法运行在subscribe()所在线程，可能不在主线程，不能刷新ui。此时我们可以使用Observable.doOnSubscribe()方法，这个方法会在订阅后被调用，在observable触发事件前调用，在其中执行ui操作，在这个方法调用后调用subscribeOn(AndroidSchedulers.mainThread()),这样Observable.doOnSubscribe()就可以运行在主线程，并且与这个observable调用了几次subscribeOn()无关，都会生效。

# 变换
> 对序列中的事件对象，或整个事件序列进行处理，转换成不同的事件序列;

- #### map()
主要将一个对象转换为另外一个对象
```
Observable.just("images/logo.png") // 输入类型 String
    .map(new Func1<String, Bitmap>() {
        @Override
        public Bitmap call(String filePath) {
            //将String类型转换为bitmap类型
            return getBitmapFromPath(filePath);
        }
    })
    .subscribe(new Action1<Bitmap>() {
        @Override
        public void call(Bitmap bitmap) { // 参数类型 Bitmap
            showBitmap(bitmap);
        }
    });
```

- #### flatMap()
- **一个对象转换为一系列对象**
发送一个对象时，会将其转换为一个Observable，这样一个对象就转换为一系列事件。而发送下一个对象，又会产生一个Observable，之后这些Observable合并在一起，统一在一个序列中发送对象，这样就实现一个对象转换为多个对象。
```
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
![](https://note.youdao.com/yws/public/resource/4fbcae57ed9c225f22300c9968c9ee8b/xmlnote/19FE073958B94EDE9A17BE6202BB3A5D/15878)

- **嵌套异步操作**
例如在需要获取接口返回的一个参数后，才能获取另一接口返回的参数，就可以使用flatMap()实现嵌套请求网络；
```
networkClient.token() // 返回 Observable<String>，在订阅时请求 token，并在响应后发送 token
    .flatMap(new Func1<String, Observable<Messages>>() {
        @Override
        public Observable<Messages> call(String token) {
            // 返回 Observable<Messages>，在订阅时请求消息列表，并在响应后发送请求到的消息列表
            return networkClient.messages();
        }
    })
    .subscribe(new Action1<Messages>() {
        @Override
        public void call(Messages messages) {
            // 处理显示消息列表
            showMessages(messages);
        }
    });
```
如上首先请求了networkClient.token()请求一个接口，这个接口会返回一个Observable<String>，而我们所需的是用这个返回的String去请求另外一个接口，因此使用flatMap拦截这个Observable，提取String再请求另外接口。而上面的操作都要等到调用subscribe才会触发，observer获取的事件对象是第二个请求接口的返回结果。