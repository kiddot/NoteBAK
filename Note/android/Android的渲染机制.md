# Android的渲染机制

## 知识储备

- **CPU** ：中央处理器,它集成了运算,缓冲,控制等单元,包括绘图功能.CPU将对象处理为多维图形,纹理
- **GPU** ： 一个类似于CPU的专门用来处理Graphics的处理器, 作用用来帮助加快栅格化操作。也具备缓存的数据
- **OpenGL ES** ： 手持嵌入式设备的3DAPI,跨平台的、功能完善的2D和3D图形应用程序接口API
- **DisplayList** ： 在Android把XML布局文件转换成GPU能够识别并绘制的对象。这个操作是在DisplayList的帮助下完成的。DisplayList持有所有将要交给GPU绘制到屏幕上的数据信息。
- **栅格化** ：是 将图片等矢量资源,转化为一格格像素点的像素图,显示到屏幕上,过程图如下.

![栅格化操作](http://upload-images.jianshu.io/upload_images/1848340-4f17720d731bc5af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



- **垂直同步VSYNC** : 让显卡的运算和显示器刷新率一致以稳定输出的画面质量。它告知GPU在载入新帧之前，要等待屏幕绘制完成前一帧。
- **Refresh Rate** ： 屏幕一秒内刷新屏幕的次数,由硬件决定,例如60Hz
- **Frame Rate** ： GPU一秒绘制操作的帧数,单位是30fps



## 渲染机制分析

### 渲染流程线

- UI对象

- CPU处理为多维图形，纹理

- 通过OpenGL ES接口调用GPU

- GPU对图进行光栅化

- 硬件时钟，垂直同步

- 投射到屏幕

  流程如下图：

  ![图片名称](http://upload-images.jianshu.io/upload_images/1848340-0224bc9dde7b163e?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 渲染时间线

- Android系统每隔16ms发出垂直信号，触发对UI的渲染，如果每次渲染成功，这样就能够达到流畅的画面所需要的60fps，这意味这计算渲染的大多数操作都必须在16ms内完成。



### 正常情况

![正常情况](http://upload-images.jianshu.io/upload_images/1848340-d5f27647c1cd1f99?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 渲染超时，即超过16ms

- 当一帧画面渲染超过16ms的时候，垂直同步机制会让显示器硬件等待GPU完成栅格化渲染操作，这样会让一帧画面多停留16ms，甚至更多，这样就造成了画面卡顿的视觉效果。

![img](http://upload-images.jianshu.io/upload_images/1848340-ef60409448b8395e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 渲染时会出现的问题

### GPU过度绘制

> GPU的绘制过程,就跟刷墙一样,一层层的进行,16ms刷一次.这样就会造成,图层覆盖的现象,即无用的图层还被绘制在底层,造成不必要的浪费.

![这里写图片描述](http://upload-images.jianshu.io/upload_images/1848340-aec8de30bcbdd39c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



### 过度绘制查看工具

> 在手机端的开发者选项里,有OverDraw监测工具,**调试GPU过度绘制工具**,
> 其中**颜色代表渲染的图层情况,分别代表1层,2层,3层,4层覆盖.**

![这里写图片描述](http://upload-images.jianshu.io/upload_images/1848340-66d89d7950737583?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)





## 计算渲染的耗时

- 任何时候View中的绘制内容发生变化时，都会重新执行创建DisplayList，渲染DisplayList，更新到屏幕上等一系列操作。这个流程的表现性能取决于你的View的复杂程度，View的状态变化以及渲染管道的执行性能。


- 当View的大小发生改变，DisplayList就会重新创建，然后再渲染，而当View发生位移，则DisplayList不会重新创建，而是执行重新渲染的操作。


- 当View过于复杂，操作又过于复杂，就会计算渲染时间超过16ms，产生卡顿问题。

![img](http://upload-images.jianshu.io/upload_images/1848340-0fe5822404e7898d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 如何优化

### Android系统本身的优化

在Android里面那些由主题所提供的资源，例如Bitmaps，Drawables都是一起打包到统一的Texture纹理当中，然后再传递到 GPU里面，这意味着每次你需要使用这些资源的时候，都是直接从纹理里面进行获取渲染的。



### 程序员自身需要做的优化

> 扁平化处理,防止过度绘制OverDraw



#### 每个layout最外层的父容器是否需要？

- 多余的布局，比如LinearLayout等，会让GPU多渲染一层图，所以能省去就省去。



#### 布局层级优化

- 查看自己的布局，深的层级，能减少层级的情况尽量减少。



#### Hierarchy Viewer工具

这是查看耗时的工具和布局树的深度的工具



![这里写图片描述](http://upload-images.jianshu.io/upload_images/1848340-a8d7fbfdceefcade?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

利用它，可以查看自己布局树，尽量减少树的深度，减少每个View的渲染时间



#### 图片选择

Android界面能用png尽量用png。因为32位png颜色过度平滑且支持透明。jpg是像素化压缩过的图片，质量已经下降了，不适合再拿来做9path的按钮和平铺拉伸的空间。



对于颜色繁杂的，**照片墙纸之类的图片（应用的启动画面喜欢搞这种），那用jpg是最好不过**了，这种图片压缩前压缩后肉眼分辨几乎不计，如果保存成png体积将是jpg的几倍甚至几十倍，严重浪费体积。



#### 清理不必要的背景

![这里写图片描述](http://upload-images.jianshu.io/upload_images/1848340-4685b8733b3f1239?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#### 当背景无法避免，尽量使用Color.TRANSPARENT

> 因为透明色`Color.TRANSPARENT`是不会被渲染的,他是透明的.



#### 优化自定义View的计算

View中的方法OnMeasure,OnLayout,OnDraw.在我们自定义View起到了决定作用,要学会研究其中的优化方法.



> 比如学会裁剪掉View的覆盖部分,增加cpu的计算量,来优化GPU的渲染











