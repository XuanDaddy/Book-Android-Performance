# 【三】AsyncLayoutInflater

### 异步Inflate

* WorkThread加载布局；
* 回调主线程；
* 节约主线程时间；

### 使用方式

使用构造方法创建,调用inflate方法，回调主线程。

```java
super.onCreate(savedInstanceState);
new AsyncLayoutInflater(this).inflate(R.layout.activity_main, null,
new AsyncLayoutInflater.OnInflateFinishedListener() {
  @Override
  public void onInflateFinished(@NonNull View view, int resid,@Nullable ViewGroup parent) {
        setContentView(view);
      }
});
```

### 源码解析

* 构建方法

```java
    public AsyncLayoutInflater(@NonNull Context context) {
        mInflater = new BasicInflater(context);
        mHandler = new Handler(mHandlerCallback);
        mInflateThread = InflateThread.getInstance();
    }
```

1. 构建mInflater，它是真正加载布局的Inflater;

   ```java
   private static class BasicInflater extends LayoutInflater {
           private static final String[] sClassPrefixList = {
               "android.widget.",
               "android.webkit.",
               "android.app."
           };
   
           BasicInflater(Context context) {
               super(context);
           }
   
           @Override
           public LayoutInflater cloneInContext(Context newContext) {
               return new BasicInflater(newContext);
           }
   
           @Override
           protected View onCreateView(String name, AttributeSet attrs) throws ClassNotFoundException {
               for (String prefix : sClassPrefixList) {
                   try {
                       View view = createView(name, prefix, attrs);
                       if (view != null) {
                           return view;
                       }
                   } catch (ClassNotFoundException e) {
                       // In this case we want to let the base class take a crack
                       // at it.
                   }
               }
   
               return super.onCreateView(name, attrs);
           }
       }	
   ```

2. 构建mHandler，向主线程发送消息回调

   由于创建AsyncLayoutInflater必须是在主线程，所以Handler不需要指定Looper，也一定是在主线程。

   Handler中接收一个Callback，用于回调主线程。

   ```java
       private Callback mHandlerCallback = new Callback() {
           @Override
           public boolean handleMessage(Message msg) {
               InflateRequest request = (InflateRequest) msg.obj;
               if (request.view == null) {
                   request.view = mInflater.inflate(
                           request.resid, request.parent, false);
               }
               request.callback.onInflateFinished(
                       request.view, request.resid, request.parent);
               mInflateThread.releaseRequest(request);
               return true;
           }
       };
   ```

   需要注意的是，callback中如果异步加载xml失败，则会使用主线程再次inflate布局，这是一种回退的机制（见if判断）。

3. 构建异步线程InflateThread

   ```java
    private static class InflateThread extends Thread {
           private static final InflateThread sInstance;
           static {
               sInstance = new InflateThread();
               sInstance.start();
           }
   
           public static InflateThread getInstance() {
               return sInstance;
           }
   
           private ArrayBlockingQueue<InflateRequest> mQueue = new ArrayBlockingQueue<>(10);
           private SynchronizedPool<InflateRequest> mRequestPool = new SynchronizedPool<>(10);
   
           // Extracted to its own method to ensure locals have a constrained liveness
           // scope by the GC. This is needed to avoid keeping previous request references
           // alive for an indeterminate amount of time, see b/33158143 for details
           public void runInner() {
               InflateRequest request;
               try {
                   request = mQueue.take();
               } catch (InterruptedException ex) {
                   // Odd, just continue
                   Log.w(TAG, ex);
                   return;
               }
   
               try {
                   request.view = request.inflater.mInflater.inflate(
                           request.resid, request.parent, false);
               } catch (RuntimeException ex) {
                   // Probably a Looper failure, retry on the UI thread
                   Log.w(TAG, "Failed to inflate resource in the background! Retrying on the UI"
                           + " thread", ex);
               }
               Message.obtain(request.inflater.mHandler, 0, request)
                       .sendToTarget();
           }
   
           @Override
           public void run() {
               while (true) {
                   runInner();
               }
           }
   
           public InflateRequest obtainRequest() {
               InflateRequest obj = mRequestPool.acquire();
               if (obj == null) {
                   obj = new InflateRequest();
               }
               return obj;
           }
   
           public void releaseRequest(InflateRequest obj) {
               obj.callback = null;
               obj.inflater = null;
               obj.parent = null;
               obj.resid = 0;
               obj.view = null;
               mRequestPool.release(obj);
           }
   
           public void enqueue(InflateRequest request) {
               try {
                   mQueue.put(request);
               } catch (InterruptedException e) {
                   throw new RuntimeException(
                           "Failed to enqueue async inflate request", e);
               }
           }
       }
   ```

   异步线程单例实现（static开启线程执行），其中包含一个队列，run方法不断从队列中取出Reqesut进行处理，Request就是数据的封装。

   runInner中也是通过mInflater加载布局，唯一不同的是它是在工作线程中执行。

   对象池，缓存对象的实现：

   ```java
   private SynchronizedPool<InflateRequest> mRequestPool = new SynchronizedPool<>(10);
   ```

   源码很简单，可以直接查看。

   **acquire方法**：从对象池取出一个对象；

   **release方法**：将对象缓存到对象池中，如果当前对象已经存在，会抛出异常；

* inflate方法

  ```java
      @UiThread
      public void inflate(@LayoutRes int resid, @Nullable ViewGroup parent,
              @NonNull OnInflateFinishedListener callback) {
          if (callback == null) {
              throw new NullPointerException("callback argument may not be null!");
          }
          InflateRequest request = mInflateThread.obtainRequest();
          request.inflater = this;
          request.resid = resid;
          request.parent = parent;
          request.callback = callback;
          mInflateThread.enqueue(request);
      }
  ```

  该方法在线程中执行，通过封装数据request，加入到的线程的队列中执行。

### 缺陷

1. 高版本特性不能兼容低版本，如AppCompat是通过对应的组件进行兼容，但是AsyncLayoutInflater并没有做相关的兼容操作；

   **如何解决兼容问题？**

   可以自定义AsyncLayoutInflater：由于AsyncLayoutInflater是final，并不能继承，所以可以直接拷贝一份AsyncLayoutInflater，然后在Inflater中进行处理（可以参考系统相应的兼容处理）。

2. 不能设置Factory；

   同样，可以自定义AsyncLayoutInflater来进行解决。