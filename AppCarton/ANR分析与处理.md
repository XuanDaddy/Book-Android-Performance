# 【三】ANR分析与处理

### ANR种类

|        名称        |     对应组件     |     超时时间      |           描述           |
| :----------------: | :--------------: | :---------------: | :----------------------: |
| KeyDispatchTimeout |     Activity     |        5s         | 按键、触摸事件未处理完成 |
|  BroadcastTimeout  | BroadcastReciver | 前台10s，后台60s  |      广播未响应完成      |
|   ServiceTimeout   |     Service      | 前台20s，后台200s |    Service未处理完成     |

### ANR常见原因

- UI Thread 进行耗时操作，如 IO 操作；
- UI Thread 进程 IPC binder 和另一个进程通信，另一个进程长时间不返回结果；
- UI Thread 和子线程竞争同一个锁，在等待子线程释放锁资源而超时；
- UI Thread 和子线程死锁；

### ANR执行流程

1. 发生ANR；
2. 进程接收异常终止信号，开始写入进程ANR信息；
3. 弹出ANR提示框（Rom表现不一）；

### ANR解决套路

* 线下ANR处理方案

  * 通过导出trace文件进行分析；

  ```shell
  adb pull data/anr/traces.txt //存储路径
  ```

  * 详细分析：CPU、IO、锁；

* 线上ANR监控方案

  * 通过FileObserver监控文件变化（**注意：高版本有权限问题，无法监控文件变化**）；

  * ANR-WatchDog

    > * 非侵入式的ANR监控组件
    > * 依赖：compile 'com.github.anrwatchdog:anrwatchdog:1.4.0'
    > * 地址：https://github.com/SalomonBrys/ANR-WatchDog

    原理来自于`消息机制`，主线程存在的死循环，会不断从消息队列中取出消息进行处理。而队列中的消息来自于其它线程。

    当主线程出现ANR时，一定是Looper中取出消息并处理时，发生了超时，而这个操作是在`dispatchMessage`这个方法中进行处理。

    所以，ANR-WatchLog在一个新线程中，不断循环，记录一个值`tick`，向主线程post这个值并赋值为0，然后线程睡眠一个阀值（默认5s），由于`tick`使用了`volatile`能够保证内存可见性，当睡眠结束后，判断`tick`不为0，则使用主线程没有处理刚刚post的消息，认为存在ANR，生成ANRError并回调。

    

    //todo 代码实例

