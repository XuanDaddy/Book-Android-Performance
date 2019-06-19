# App内存优化

### 内存问题

* 内存抖动：锯齿状、GC导致卡顿
* 内存泄露：可用内存减少、频繁GC
* 内存溢出：OOM、程序异常

### 工具选择

* Memory Profiler
  * AndroidStudio自带工具，实时图表展示应用内存使用量
  * 识别内存泄露、抖动等
  * 提供捕获堆转储、强制GC以及跟踪内存分配的能力
* Memory Analyzer
  * 强大的Java Heap分析工具，查找内存泄露及内存占用
  * 生成整体报告、分析问题等
  * 线下深入使用
* LeakCanary
  * 自动内存泄露检测
  * 线下集成(线上会影响性能)

### 整体流程

* 项目中集成`LeakCanary`进行内存泄漏检测
* 有内存泄露后使用AS的profiler工具进行分析并获取到`.hprof`文件
* 使用`hprof-conv `转换文件为MAT的标准文件
* 使用`MAT`打开文件进行分析

### 优化细节

* LargeHeap属性

  > 我们知道安卓系统对于每个应用都有内存使用的限制，机器的内存限制，在/system/build.prop文件中配置的。
  >
  > ```properties
  > dalvik.vm.heapsize=128m  
  > dalvik.vm.heapgrowthlimit=64m 
  > ```

这里，heapgrowthlimit就是一个普通应用的内存限制，用ActivityManager.getLargeMemoryClass()获得的值就是这个。而heapsize是在manifest中设置了largeHeap=true之后，可以使用最大内存值。
设置largeHeap的确可以增加内存的申请量。但不是系统有多少内存就可以申请多少，而是由dalvik.vm.heapsize限制。

* onTrimMemory

  > 主动处理一些事情，会影响用户体验。

* 使用更优集合：SparseArray



