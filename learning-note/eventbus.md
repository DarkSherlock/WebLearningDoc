EventBUs注解：

    @Documented
    @Retention(RetentionPolicy.RUNTIME) // 注解不仅被保存到class文件中，jvm加载class文件之后，仍然存在
    @Target({ElementType.METHOD})  // 作用在方法上
    public @interface Subscribe {
    
    	// 指定事件订阅方法所在的线程模式，也就是决定订阅方法是在哪个线程，默认是POSTING模式
    	ThreadMode threadMode() default ThreadMode.POSTING;
    
    	// 是否支持粘性事件
    	boolean sticky() default false;
    
    	// 优先级，如果指定了优先级，则若干方法接收同一事件时，优先级高的方法会先接收到。
    	int priority() default 0;
    }

**ThreadMode**可以指定的模式有：

**ThreadMode.POSTING**：默认的线程模式，在哪个线程发送事件就在对应线程处理事件，避免了线程切换，效率高。

**ThreadMode.MAIN**：如在主线程（UI线程）发送事件，则直接在主线程处理事件；如果在子线程发送事件，则先将事件入队列，然后通过 Handler 切换到主线程，依次处理事件。

**ThreadMode.MAIN_ORDERED**：无论在哪个线程发送事件，都将事件加入到队列中，然后通过Handler切换到主线程，依次处理事件。

**ThreadMode.BACKGROUND**：与ThreadMode.MAIN相反，如果在子线程发送事件，则直接在子线程处理事件；如果在主线程上发送事件，则先将事件入队列，然后通过线程池处理事件。

**ThreadMode.ASYNC**：与ThreadMode.MAIN_ORDERED相反，无论在哪个线程发送事件，都将事件加入到队列中，然后通过线程池执行事件


## 注册过程 ：

传入当前注册对象 利用反射获取所有方法，然后取出其中带有Subsribe注解 并且是public修饰方法入参只有一个的方法，然后将这些方法信息封装成一个SubsribeMethod，然后把这些SubscribeMethod和传入的注册对象封装成Subcribetion对象，然后根据evnttype将这些Subscription分组，存入名为subscriptionsByEventType的map中（key 为event， value 为List，因为eventtype可能对应多个方法所以value为list），，再创建名为 typesBySubscriber的map， 注册类对象为key , list为value（注册对象可能有多个eventType)。 

## 解除注册
传入解除注册对象，去typesBySubcriber的Map中找出当前注册对象所包含的evntType list，然后遍历这个list去subcribptionsByType的map中取出来eventType所对应的包含所有subscriptions的list，然后遍历这个list将subcriptions对应subscriber是当前解除对象就将其移除

## 发送事件
post也不难，首先是将发送的事件保存在postingState中的队列里面，它是线程独有的，然后遍历postingState中的事件队列，拿出该线程下，所有的事件的集合，然后遍历它，再根据subscriptionsByEventType，取出该事件所对应的所有订阅方法，然后看是否能够直接处理，如果能，直接反射调用订阅方法，如果不能，直接通过HandlerPower、BackgroundPower、AsyncPower切换线程后，再进行反射调用处理。
其中HandlerPower内部就直是封装了个Handler，每次调用的时候，先将事件加入到队列中，然后根据Handler切换到主线程，按顺序取出队列中的事件，反射执行
BackgroundPower是封装了catchThreadPool用于执行任务,  AsyncPower与它类似，但是里面没有同步锁，每次执行都会新开辟一个子线程去执行任务。而BackgroundPower之会开一个线程

## stick事件
如果需要发送粘性事件的话，在发送的时候，会将粘性事件的事件类型和对应事件存储到map stickyEvents中，等新的注册类进行注册的时候，如果有的订阅方法支持粘性事件，则会在注册的时候，取出stickyEvents里面的存储粘性事件，然后遍历将对应类型的事件在发送给他处理。



参考：https://juejin.im/post/5da97188e51d4524a21c481f


