---
title: HandlerThread相关
date: 2019-10-12 12:58:35
tags:
	- blog
	- HandlerThread
---

不管是子线程发还是主线程发送消息，接收都在子线程，内部持有looper,不调用其 quit()接口，就一直存在

## 子线程和子线程之间的通信：
![图片](https://raw.githubusercontent.com/tuyrt7/tuyrt7.github.io/master/uploads/img/handler-thread.png)


<!--more-->

很明显的一点就是，我们要在子线程中调用Looper.prepare() 为一个线程开启一个消息循环，默认情况下Android中新诞生的线程是没有开启消息循环的。（主线程除外，主线程系统会自动为其创建Looper对象，开启消息循环。） Looper对象通过MessageQueue来存放消息和事件。一个线程只能有一个Looper，对应一个MessageQueue。 然后通过Looper.loop() 让Looper开始工作，从消息队列里取消息，处理消息。

> 注意：写在Looper.loop()之后的代码不会被执行，这个函数内部应该是一个循环，当调用mHandler.getLooper().quit()后，loop才会中止，其后的代码才能得以运行。

然而这一切都可以用HandlerThread类来帮我们做这些逻辑操作。

### HandlerThread  

HandlerThread本质上就是一个普通Thread,只不过内部建立了Looper.

### HandlerThread的常规用法  

- 创建一个HandlerThread 

`mThread = new HandlerThread("handler_thread");`

- 启动一个HandlerThread

`mThread.start();`

- 退出循环 


Looper是通过调用loop方法驱动着消息循环的进行: 从MessageQueue中阻塞式地取出一个消息，然后让Handler处理该消息，周而复始，loop方法是个死循环方法。


那如何终止消息循环呢？我们可以调用Looper的quit方法或quitSafely方法，二者稍有不同。 

```
  
 	/**
     * Quits the looper.
     * <p>
     * Causes the {@link #loop} method to terminate without processing any
     * more messages in the message queue.
     * </p><p>
     * Any attempt to post messages to the queue after the looper is asked to quit will fail.
     * For example, the {@link Handler#sendMessage(Message)} method will return false.
     * </p><p class="note">
     * Using this method may be unsafe because some messages may not be delivered
     * before the looper terminates.  Consider using {@link #quitSafely} instead to ensure
     * that all pending work is completed in an orderly manner.
     * </p>
     *
     * @see #quitSafely
     */
    public void quit() {
        mQueue.quit(false);
    }

    /**
     * Quits the looper safely.
     * <p>
     * Causes the {@link #loop} method to terminate as soon as all remaining messages
     * in the message queue that are already due to be delivered have been handled.
     * However pending delayed messages with due times in the future will not be
     * delivered before the loop terminates.
     * </p><p>
     * Any attempt to post messages to the queue after the looper is asked to quit will fail.
     * For example, the {@link Handler#sendMessage(Message)} method will return false.
     * </p>
     */
    public void quitSafely() {
        mQueue.quit(true);
    } 
```


#### 相同点： 

将不在接受新的事件加入消息队列。

#### 不同点 

当我们调用Looper的quit方法时，实际上执行了MessageQueue中的removeAllMessagesLocked方法，该方法的作用是把MessageQueue消息池中所有的消息全部清空，无论是延迟消息（延迟消息是指通过sendMessageDelayed或通过postDelayed等方法发送的需要延迟执行的消息）还是非延迟消息。
 
当我们调用Looper的quitSafely方法时，实际上执行了MessageQueue中的removeAllFutureMessagesLocked方法，通过名字就可以看出，该方法只会清空MessageQueue消息池中所有的延迟消息，并将消息池中所有的非延迟消息派发出去让Handler去处理，quitSafely相比于quit方法安全之处在于清空消息之前会派发所有的非延迟消息。 

无论是调用了quit方法还是quitSafely方法只会，Looper就不再接收新的消息。即在调用了Looper的quit或quitSafely方法之后，消息循环就终结了，这时候再通过Handler调用sendMessage或post等方法发送消息时均返回false，表示消息没有成功放入消息队列MessageQueue中，因为消息队列已经退出了。 

需要注意的是Looper的quit方法从API Level 1就存在了，但是Looper的quitSafely方法从API Level 18才添加进来。

小例子： 
  
```   

	package com.app;
	import android.os.Bundle;
	import android.os.Handler;
	import android.os.HandlerThread;
	import android.os.Message;
	import android.support.v7.app.AppCompatActivity;
	import android.util.Log;
	public class MainActivity extends AppCompatActivity {

    private HandlerThread myHandlerThread ;
    private Handler handler ;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        //创建一个线程,线程名字：handler-thread
        myHandlerThread = new HandlerThread( "handler-thread") ;
        //开启一个线程
        myHandlerThread.start();
        //在这个线程中创建一个handler对象
        handler = new Handler( myHandlerThread.getLooper() ){
            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                //这个方法是运行在 handler-thread 线程中的 ，可以执行耗时操作
                Log.d( "handler " , "消息： " + msg.what + "  线程： " + Thread.currentThread().getName()  ) ;

            }
        };

        //在主线程给handler发送消息
        handler.sendEmptyMessage( 1 ) ;

        new Thread(new Runnable() {
            @Override
            public void run() {
             //在子线程给handler发送数据
             handler.sendEmptyMessage( 2 ) ;
            }
        }).start() ;

    }

    @Override
    protected void onDestroy() {
        super.onDestroy();

        //释放资源
        myHandlerThread.quit() ;
    } }  
  

``` 


运行效果：  

> handler: 消息： 1  线程： handler-thread  
> handler: 消息： 2  线程： handler-thread


### HandlerThread的特点
HandlerThread将loop转到子线程中处理，说白了就是将分担MainLooper的工作量，降低了主线程的压力，使主界面更流畅。

开启一个线程起到多个线程的作用。处理任务是串行执行，按消息发送顺序进行处理。HandlerThread本质是一个线程，在线程内部，代码是串行处理的。

但是由于每一个任务都将以队列的方式逐个被执行到，一旦队列中有某个任务执行时间过长，那么就会导致后续的任务都会被延迟处理。

HandlerThread拥有自己的消息队列，它不会干扰或阻塞UI线程。

对于网络IO操作，HandlerThread并不适合，因为它只有一个线程，还得排队一个一个等着。

参考文章：  
Android Handler、Loop 的简单使用 http://www.cnblogs.com/zhaoyanjun/p/5546843.html  

Android 如何有效的解决内存泄漏的问题 http://www.cnblogs.com/zhaoyanjun/p/5981386.html 

Android 更新UI的几种方式 http://www.cnblogs.com/zhaoyanjun/p/5546683.html
