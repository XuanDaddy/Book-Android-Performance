# App布局优化

### 绘制原理

* CPU负责计算显示内容
* GPU负责栅格化(UI元素绘制到屏幕上)
* 16ms发出VSync信号触发UI渲染

### 优化工具

* Systrace

  * 关注Frames
  * 正常：绿色圆点，丢帧：黄色或红色
  * Alert栏

  ![image](./images/Systrace-Frame.png)

  ![image](./images/Systrace-Alert.png)

* Layout Inspector

  > AS自带工具，可查看视图层次结构

![image](./images/Layout-Inspector.png)

* Choreographer

  * 获取FPS，线上使用，具备实时性
  * Api16之后才能使用

  ```java
  Choreographer.getInstance().postFrameCallback();
  ```

  