---
layout:     post
title:      "RxJava 浅析"
subtitle:   "RxJava使用及操作符介绍"
date:       2016-02-25
author:     "张志超"
header-img: "img/post-bg-android.jpg"
tags:
    - Android
    - 分享
    - Blog
---

# Rxjava-learn

## 1、github 项目地址：  https://github.com/ReactiveX/RxJava
（RxAndroid 是一个专门为Android开发做出最小适配框架，可以方便的写出响应式的组件和一站式服务）

## 2、rxjava是一个响应式的编程框架，基于观察者模式。
其中关键的要素就是Observable(事件产生者,也是事件的观察者)  Operator（操作符） Subscriber（事件的消费者，也是事件的订阅者）

特点：使用可观察序列编写异步和事件驱动的库，扩展了观察者模式以支持数据和事件序列，并且加入了operator操作符来实现数据转换和事件过滤

1. 易于并发从而更好的利用服务器的能力      （可以自由指定运行的线程）
2. 易于有条件的异步执行                               （通过操作符控制流程）
3. 一种更好的方式来避免回调地狱                 （多个事件源基于组合而不是嵌套）
4. 一种响应式方法                                          （编码更简单）

优点：将开发者的注意力从低级别的线程同步、线程安全、并发数据对象这些问题中转移

## 3、一个简单的示例

#### A、创建一个Observable
    Observable<String> myObservable = Observable.create(
        new Observable.OnSubscribe<String>() {
             @Override
            public void call(Subscriber<? super String>sub) {
            sub.onNext("Hello, world!");
            sub.onCompleted();
            }
        });

#### B、创建一个Subscriber
    Subscriber<String> mySubscriber = new Subscriber<String>() {
        @Override
        public void onNext(String s) {
            System.out.println(s);
        }
        @Override
        public void onCompleted() {
        }
        @Override
        public void onError(Throwable e) {
        }
    };

#### C、将两者联系起来
    myObservable.subscribe(mySubscriber); // Outputs "Hello, world!"

这是一个展示流程的例子：myObservable是事件源，mySubscriber是订阅者，
通过Observable的subscribe方法，将事件输出给订阅者去消费。

## 4、什么是Observable   Observer   Subscriber   Subscription
   
Observable  事件观察者或者事件生产者  这两种叫法是针对不同的对象而言的。第一：对于Subscriber它是事件的生产者，因为当使用subscribe方法对一个Observable添加一个订阅者的时候，这个时候会立即调用onSubscribe的call方法，将产生的一个字符串“Hello，world”这个事件交给订阅者的onNext()方法。第二：对于产生的这个字符串“Hello， world”而言，Observable就是一个事件观察者，它观察到了这个字符串的产生，然后将这个字符串产生的事件发送给了订阅者。

Observer和Subscriber是一个东西，Subscriber继承自Observer。

Subscription是一个接口，提供了对一个Subscriber进行取消订阅（unSubscribe）和是否取消订阅（isUnsubscribe）的功能，它的具体实现就是Subscriber。上例中的第三步中subscribe方法将会返回一个Subscription，用户可以方便的取消订阅。

## 5、其他的创建Observable的方式
        Observable.From(DataCollection) 使用一个数据集创建一个Observable，自动遍历发射集合中每条数据
        Observable.just(a java method return )  根据一个或多个其他的方法返回值创建一个Observable，因为just可以接受1-9各参数，然后按照参数顺序发射他们。也可以是一个数据集合，像from方法，但他是发射整个列表。

## 6、Subject = Observable + Observer （既是事件产生者，又是事件订阅者）
     Subject有四种类型 PublishSubject  BehaviorSubject     ReplaySubject     AsyncSubject

PublishSubject 是一个可以在任何时候发射事件的事件产生者，而不一定是在订阅者开始订阅的时候。
BehaviorSubject会首先向他的订阅者发送截至订阅前最新的一个数据对象（或初始值），然后正常发送订阅后的数据流。
ReplaySubject会缓存它所订阅的所有数据，向任意一个订阅它的观察者重发。
AsyncSubject只会发布最后一个数据 给已经订阅的每一个观察者。

***

最最关键的几个概念：**Observable   Observer    Action(Observer observer)

当Observable被subscribe（订阅）的时候，调用action的call方法

和观察者模式对比：  被观察者     观察者      被观察持有观察者的引用，当数据变化时通知观察者

***


## 7、操作符
- 
   __repeat()  对一个Observable重复发射数据__例：

    Observable.just(1, 2).repeat(5).subscribe(new Subscriber<Integer>() {
        @Override
        public void onCompleted() {
            
        }
            
        @Override
        public void onError(Throwable e) {
            
        }
            
        @Override
        public void onNext(Integer integer) {
            System.out.println("integer------>" + integer);
        }
    });
    
- 
  __defer() 延迟Observable的创建直到观察者订阅__例

    private Observable<Long> getDeferObservable() {
        return Observable.defer(new Func0<Observable<Long>>() {
            @Override
            public Observable<Long> call() {
                return getJustObservable();
            }
        });
    }

__每次生成新的observable__



    @Test
    public void testDefer() {
        Observable<Long> observable = getDeferObservable();
        for (int i = 0; i < 10; i++) {
            observable.subscribe(new Action1<Long>() {
                @Override
                public void call(Long aLong) {
                    System.out.println("aLong------>" + aLong);
                }
            });
        }
    }

- 
__interval() 在指定的时间间隔内重复数字 0到正无穷__

    Subscription topeMePlease = Observable.interval(3, TimeUnit.SECONDS)
            .subscribe(new Observer<Long>() {
                @Override
                public void onCompleted() {
                
                }
                
                @Override
                public void onError(Throwable e) {
                
                }
                
                @Override
                public void onNext(Long aLong) {
                    System.out.println("aLong---->"+aLong);
                }
            });

- 
__timer()  指定延迟时间指定间隔发射__

    Observable.timer(3, 100, TimeUnit.MILLISECONDS).subscribe(new Action1<Long>() {
        @Override
        public void call(Long aLong) {
            System.out.println("aLong------>" + aLong);
        }
    });

- 
__filter()  过滤出符合要求的数据__

    filter((appInfo) -> appInfo.getName().startWith(“C”)) //过滤出C开头的应用名称

- 
__take()  指定原始序列中的前几条数据发射__

    take(3)
    
- 
__takeLast()  指定原始序列中的最后几条数据发射__

    takeLast(3)
    
- 
__distinct()  去除重复数据  可以用来防止界面控件重复点击__

- 
__distinctUntilChanged()  去除与上一个重复的值__

- 
__first()和last() 发射原始序列中的第一个或最后一个值__

-
__firstOrDefault()和lastOrDefault()  当观测序列完成时发送默认值__

- 
__skip()和skipLast()   不发射前N个值或者后N个值__

- 
__ElementAt()   elementAtOrDefault()     发射指定位置的元素 ，如果没有就发送默认值__

- 
__sample(30, TimeUnit.SECONDS)  在指定时间间隔内由Observable发射最近一次的数值  再加一个throttleFirst()就是发射第一个而不是最近一个元素__

- 
__timeout()   每隔一定时间发射至少一次数据，如果在指定时间间隔内没有得到一个值则发送一个错误__

- 
__debounce()   过滤掉由Observable发射的速率过快的数据__

- 
__map  指定一个fun对象，然后将它应用到每一个由Observable发射的值上__

- 
__flatMap()  根据上一个Observable发射的数据生成新的Observable，注意新产生的Observable是平铺的，也就是说最终得到数据顺序是不定的，并且有一个产生error，此次调用就会结束__

- 
__concatMap()   解决的flatMap()的交叉问题，能够把发射的值连续在一起，而不是合并他们__

- 
__scan()   累加器  对原始Observable发射的每一项数据都应用一个函数，计算出函数的结果值，并将该值填回可观测序列，等待和下一次发射的数据一起使用。__例：

    Observable.just(1,2,3,4,5)
                .scan((sum, item) -> sum + item)
                .subscribe(new Subscriber<Interger> () {
        @Override
        public void onCompleted() {
            
        }
        
        @Override
        public void onError(Throwable e) {
        
        }
        
        @Override
        public void onNext(Integer integer) {
            System.out.println("integer------>" + integer);
        }
    });

__输出结果为：1   3  6  10  15__
（这个操作符可用来对数据进行排序）


- 
__groupBy()  将原Observable变换成哼一个发射Observables的新的Observable。他们中的每一个新的Observable都发射一组指定的数据__

- 
__buffer()    将原Observable变换一个新的Observable，这个新的Observable每次发射一组列表而不是一个个发射__

- 
__merge()   多个Observable合并成一个最终发射的Observable  （多个Observable发射的数据类型一般相同）__

- 
__zip   合并多个Observable数据，生成新的数据__

## 8、调度器

RxJava提供了5种调度器：
    __.io()  .computation()  .immediate()  .newThread()  .trampoline()__
        
- 
Schedulers.io()  专用于io操作，但是大量的io操作会创建多个线程并占用内存
   
- 
Schedulers.computation()  计算工作默认的调度器，与io无关
   
- 
Schedulers.immedidate()  在当前线程立即执行指定的工作
   
- 
Schedulers.newThread()   为指定任务启动一个新的线程
   
- 
schedulers.tramppline()  把要执行的任务加入到当前线程任务队列中，调度器会顺序执行队列中的任务
   
   
Executors.newScheduledThreadPool(1, threadFactory);
ScheduledExecutorService
    
SubscribeOn(Schedulers.io())  指定任务工作线程
ObserveOn(AndroidSchedulers.mainThread())  指定观察者处理返回结果所在线程为ui线程
   
## 9、在Android中使用场景

(1). 先检查本地是否有数据缓存，有的话直接返回，没有的话再请求网路数据  
    对应操作符为  contact(Observable1, Observable2 ...)
    
(2). 多个接口并发请求，等所有结果返回再统一刷新页面   
    这种情况需要分两种条件：

    a、不同接口返回数据格式相同，不需要做类型判断和转换，可以用merge(Observable1, Observable2 ...)
        
    b、不同接口返回数据格式不同，需要经过处理再合并成新的数据结构，可以用
        zip(Observable1, Observable2,                                     
        new Fun2<firstResult, SecondResult, newResult>) 
        或combineLatest(Observable1, Observable2, new                                         
        Fun2<firstResult, SecondResult, newResult>)
        对于combineLatest和zip，在网络请求的使用情景下，Observable只发射一次数据，二者是没有区别的。
        如果是Observable多次发射数据的话，combineLatest会有对不同实际发射出的事件的合并有不同的合并结果。
        而zip则是一一对应的。
        
        Observable1  1  2  3
        Observable2  1  2  3 

    
(3). 一个任务的执行依赖上一个任务的返回结果,
    对应操作符为flatmap(object,Observable)，根据上一个任务的返回结构再次生成新的Observable
    
(4). 界面按钮防止连续点击，对应操作符为throttleFisrt(时间段， 时间单位)，在指定时间段内只发送一次数据
    
(5). 替代Handler实现定时器的操作符  timer(delaytime, time, timeUnit)  X秒后执行某操作
    
(6). 替代Handler.postDelay实现文本搜索的操作符为debounce(400, TimeUnit.MILLISECONDS)
    
(7). 替代Handler.postDelay实现倒计时的操作符为interval(1, TimeUnit.SECONDS) ，每隔1秒发射一次事件
    
(8). 使用schedulePeriodically做轮询请求

        Observable.create(new Observable.OnSubscribe<String>() {  
            @Override  
            public void call(final Subscriber<? super String> observer) {  
                Schedulers.newThread().createWorker()  
                      .schedulePeriodically(new Action0() {  
                          @Override  
                          public void call() {  
                              observer.onNext(doNetworkCallAndGetStringResult());  
                          }  
                      }, INITIAL_DELAY, POLLING_INTERVAL, TimeUnit.MILLISECONDS);  
            }  
        }).subscribe(new Action1<String>() {  
            @Override  
            public void call(String s) {  
                log.d("polling….”));  
            }  
        })  
    
(9)、注册界面信息填写完整，下一步操作按钮才点亮 combineLatest

        Observable<CharSequence> _emailChangeObservable = RxTextView.textChanges(_email).skip(1);  
        Observable<CharSequence> _passwordChangeObservable = RxTextView.textChanges(_password).skip(1);  
        Observable<CharSequence>   _numberChangeObservable = RxTextView.textChanges(_number).skip(1);  
        
        Observable.combineLatest(_emailChangeObservable,  
              _passwordChangeObservable,  
              _numberChangeObservable,  
              new Func3<CharSequence, CharSequence, CharSequence, Boolean>() {  
                  @Override  
                  public Boolean call(CharSequence newEmail,  
                                      CharSequence newPassword,  
                                      CharSequence newNumber) {  
                                      
                      Log.d("xiayong",newEmail+" "+newPassword+" "+newNumber);  
                      boolean emailValid = !isEmpty(newEmail) &&  
                                           EMAIL_ADDRESS.matcher(newEmail).matches();  
                      if (!emailValid) {  
                          _email.setError("Invalid Email!");  
                      }  
                      
                      boolean passValid = !isEmpty(newPassword) && newPassword.length() > 8;  
                      if (!passValid) {  
                          _password.setError("Invalid Password!");  
                      }  
                      
                      boolean numValid = !isEmpty(newNumber);  
                      if (numValid) {  
                          int num = Integer.parseInt(newNumber.toString());  
                          numValid = num > 0 && num <= 100;  
                      }  
                      if (!numValid) {  
                          _number.setError("Invalid Number!");  
                      }  
                      
                      return emailValid && passValid && numValid;  
                      
                  }  
              })//  
              .subscribe(new Observer<Boolean>() {  
                  @Override  
                  public void onCompleted() {  
                      log.d("completed");  
                  }  
                  
                  @Override  
                  public void onError(Throwable e) {  
                     log.d("Error");  
                  }  
                  
                  @Override  
                  public void onNext(Boolean formValid) {  
                     _btnValidIndicator.setEnabled(formValid);    
                  }  
              });  
    
(10)、取缓存同时取网络数据，然后更新。 ？？？
    
## 10. 源码剖析
1.

    protected Observable(OnSubscribe<T> f) {
        this.onSubscribe = f;
    }
    
Observable的构造方法，即保存构造方法中的参数OnSubscribe

2. 



    public static interface OnSubscribe<T> extends Action1<Subscriber<? super T>> {
        // cover for generics insanity
    }


OnSubscribe是一个带一个参数的Action1，它的参数是一个Subscriber

    public interface Action1<T1> extends Action {
        public void call(T1 t1);
    }

Action1中有一个call方法，其中的参数就是就是第二步创建的Subscriber

3.

    Observable observable = Observable.create(new Observable.OnSubscribe<ShopList>() {
        @Override
        public void call(Subscriber<? super ShopList> subscriber) {
            ShopList discountShops = companyRepository.getPayBillShops(offset, pageSize, regionId, longitude, latitude);
            subscriber.onNext(discountShops);
            subscriber.onCompleted();
        }
    });
    
在创建Observable的时候，传入了一个新建的OnSubscribe，然后再OnSubscribe中的call方法中，调用了call方法的参数（Subscriber）的onNext() onCompleted() 方法！！！

__注：此时的Subscriber（订阅者）并不知道是谁。__
    
至此，被观察者已经基本创建完成，这个被观察者是一个Action，这个Action的具体动作是从网络获取数据。
那么，当Action动作完成，会把结果传递给不知道是谁的一个订阅者。。。
    
4. 订阅者的创建


    public final Subscription subscribe(Subscriber<? super T> subscriber) {
        // validate and proceed
        if (subscriber == null) {
            throw new IllegalArgumentException("observer can not be null");
        }
        if (onSubscribe == null) {
            throw new IllegalStateException("onSubscribe function can not be null.");
            /*
             * the subscribe function can also be overridden but generally that's not the appropriate approach
             * so I won't mention that in the exception
             */
        }
            
        // new Subscriber so onStart it
        subscriber.onStart();
            
        /*
         * See https://github.com/ReactiveX/RxJava/issues/216 for discussion on "Guideline 6.4: Protect calls
         * to user code from within an Observer"
         */
        // if not already wrapped
        if (!(subscriber instanceof SafeSubscriber)) {
            // assign to `observer` so we return the protected version
            subscriber = new SafeSubscriber<T>(subscriber);
        }
            
        // The code below is exactly the same an unsafeSubscribe but not used because it would add a sigificent depth to alreay huge call stacks.
        try {
            // allow the hook to intercept and/or decorate
            hook.onSubscribeStart(this, onSubscribe).call(subscriber);
            return hook.onSubscribeReturn(subscriber);
        } catch (Throwable e) {
            // special handling for certain Throwable/Error/Exception types
            Exceptions.throwIfFatal(e);
            // if an unhandled error occurs executing the onSubscribe we will propagate it
            try {
                subscriber.onError(hook.onSubscribeError(e));
            } catch (OnErrorNotImplementedException e2) {
                // special handling when onError is not implemented ... we just rethrow
                throw e2;
            } catch (Throwable e2) {
                // if this happens it means the onError itself failed (perhaps an invalid function implementation)
                // so we are unable to propagate the error correctly and will just throw
                RuntimeException r = new RuntimeException("Error occurred attempting to subscribe [" + e.getMessage() + "] and the again while trying to pass to onError.", e2);
                // TODO could the hook be the cause of the error in the on error handling.
                hook.onSubscribeError(r);
                // TODO why aren't we throwing the hook's return value.
                throw r;
            }
            return Subscriptions.unsubscribed();
        }
    }
    
    关键代码:
__hook.onSubscribeStart(this, onSubscribe).call(subscriber);__
__hook.onSubscribeStart(this, onSubscribe)返回的就是Observable创建时构造方法中的参数OnSubcribe__
    
 
然后调用onSubscribe的call方法，参数就是我们subscribe方法中的参数Subscriber，接下来就一目了然了，第三步中那个不知道是谁的订阅者，就是通过subscribe方法传入的订阅者。
至此，订阅者和观察就联系起来了。
 
## 11、 多个订阅者的两种实现方法

 a、使用PublishSubject

    PublishSubject<String> stringPublishSubject = PublishSubject.create();
        Subscriber subscriber1 = new Subscriber() {
            @Override
                public void onCompleted() {
                 
                }
                    
                @Override
                public void onError(Throwable e) {
                    
                }
                    
                @Override
                public void onNext(Object o) {
                    System.out.println("subscriber1---->" + o.toString());
                }
            };
            
        Subscriber subscriber2 = new Subscriber() {
            @Override
            public void onCompleted() {
                
            }
                
            @Override
            public void onError(Throwable e) {
                
            }
            
            @Override
            public void onNext(Object o) {
                System.out.println("subscriber2---->" + o.toString());
            }
        };
        stringPublishSubject.subscribe(subscriber1);
        stringPublishSubject.subscribe(subscriber2);
        stringPublishSubject.onNext("a");

b、使用ConnectableObservable

     ConnectableObservable<String> stringConnectableObservable = getMemoryObservable().publish();
     
        Subscriber subscriber1 = new Subscriber() {
            @Override
            public void onCompleted() {
            
            }
            
            @Override
            public void onError(Throwable e) {
            
            }
            
            @Override
            public void onNext(Object o) {
                System.out.println("subscriber1---->" + o.toString());
                System.out.println("subscriber1---->" + System.currentTimeMillis());
            }
        };
        
        Subscriber subscriber2 = new Subscriber() {
            @Override
            public void onCompleted() {
            
            }
            
            @Override
            public void onError(Throwable e) {
            
            }
            
            @Override
            public void onNext(Object o) {
                System.out.println("subscriber2---->" + o.toString());
                System.out.println("subscriber2---->" + System.currentTimeMillis());
            }
        };
        
        stringConnectableObservable.subscribe(subscriber1);
        stringConnectableObservable.subscribe(subscriber2);
        
        stringConnectableObservable.connect();

    
## 12、操作符使用原理
    关键方法：Observable lift(Operator)
    

        public final <R> Observable<R> lift(final Operator<? extends R, ? super T> lift) {
            return new Observable<R>(new OnSubscribe<R>() {
                @Override
                public void call(Subscriber<? super R> o) {
                    try {
                        Subscriber<? super T> st = hook.onLift(lift).call(o);
                        try {
                            // new Subscriber created and being subscribed with so 'onStart' it
                            st.onStart();
                            onSubscribe.call(st);
                        } catch (Throwable e) {
                            // localized capture of errors rather than it skipping all operators 
                            // and ending up in the try/catch of the subscribe method which then
                            // prevents onErrorResumeNext and other similar approaches to error handling
                            if (e instanceof OnErrorNotImplementedException) {
                                throw (OnErrorNotImplementedException) e;
                            }
                            st.onError(e);
                        }
                    } catch (Throwable e) {
                        if (e instanceof OnErrorNotImplementedException) {
                            throw (OnErrorNotImplementedException) e;
                        }
                        // if the lift function failed all we can do is pass the error to the final Subscriber
                        // as we don't have the operator available to us
                        o.onError(e);
                    }
                }
            });
        }


