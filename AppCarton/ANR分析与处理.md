# 【三】ANR分析与处理

### ANR种类

|        名称        |     对应组件     |     超时时间      |           描述           |
| :----------------: | :--------------: | :---------------: | :----------------------: |
| KeyDispatchTimeout |     Activity     |        5s         | 按键、触摸事件未处理完成 |
|  BroadcastTimeout  | BroadcastReciver | 前台10s，后台60s  |      广播未响应完成      |
|   ServiceTimeout   |     Service      | 前台20s，后台200s |    Service未处理完成     |

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

    //todo 原理及代码实例

