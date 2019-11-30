# App瘦身优化

### 为什么需要做瘦身优化

* 下载转换率
* 渠道合作商要求

### Apk组成结构

* 代码相关：**classes.dex**
* 资源相关：**res、asserts、resources.arsc**
* So相关：**lib**

### Apk分析

* ApkTool，反编译工具
* Analyze APK，AS自带
  * 可查看Apk组成、大小、占比等
  * 查看Dex文件组成
  * 可对比Apk
* android-classyshark，二进制检查工具
  * [Github传送门](https://github.com/google/android-classyshark)
  * 支持多种格式：Apk、Jar、Class、So等
  * 打开该jar，直接把apk拖入