# Activity

### 一、 activity的生命周期

#### 正常生命周期

##### 1.正常打开单个Activity，然后退出应用：

这种情况是最普通的状况，Activity的生命周期会按照上图从上到下的方式走。即：onCreate --> onStart --> onResume --> 运行--> 按返回键结束程序--> onPause-->onStop-->onDestory

##### 2.打开一个Activity A，然后再打开另一个Activity B

对于A：onCreate --> onStart --> onResume --> A运行 --> A发出打开B的Intent --> onPause-->B可见-->onStop
此时，会打开B，B同样会经历一个完整的Activity生命周期。等B结束，A再度可见的时候，A会经历：onRestart-->onStart-->onResume

> 注意：B这个Activity是在A的onPause执行后才变成可见状态的，所以为了不影响B的显示，最好不要在onPause里执行一些耗时操作，可以考虑将这些操作放到onStop里，这时B已经可见了。

#### 异常情况生命周期

##### 情况1.资源相关的系统配置发生改变

资源相关的系统配置发生改变，举个例子。当前Activity处于竖屏状态的时候突然转成横屏，系统配置发生了改变，Activity就会销毁并且重建，其onPause, onStop, onDestory均会被调用。因为实在异常情况下终止的，所以系统会调用**onSaveInstanceState**来保存当前Activity状态。这个方法是在onStop之前，与onPause没有固定的时序关系。当Activity重建的时候系统会把onSaveInstanceState所保存的Bundle作为对象传递给onRestoreInstanceState和onCreate方法。

##### 情况2：资源内存不足导致低优先级Activity被杀死

Activity优先级
前台Activity——正在和用户交互的Activity，优先级最高
可见但非前台Activity——Activity中弹出的对话框导致Activity可见但无法交互
后台Activity——已经被暂停的Activity，优先级最低
系统内存不足是，会按照以上顺序杀死Activity，并通过onSaveInstanceState和onRestoreInstanceState这两个方法来存储和恢复数据。



### onSaveInstanceState

  onSaveInstanceState在onStop之前调用，但不一定在onPause之前或者之后。onRestoreInstanceState在onStart之后调用

 需要注意的是，onSaveInstanceState方法只会在Activity被异常终止，在Activity即将被销毁且**有机会重新显示的情况下**才会调用

可以总结有下面几种情况

```
1、当用户按下HOME键时。

这是显而易见的，系统不知道你按下HOME后要运行多少其他的程序，自然也不知道activity A是否会被销毁，故系统会调用onSaveInstanceState，让用户有机会保存某些非永久性的数据。以下几种情况的分析都遵循该原则

2、长按HOME键，选择运行其他的程序时。

3、按下电源按键（关闭屏幕显示）时。

4、从activity A中启动一个新的activity时。

5、屏幕方向切换时，例如从竖屏切换到横屏时。（如果不指定configchange属性） 在屏幕切换之前，系统会销毁activity A，在屏幕切换之后系统又会自动地创建activity A，所以onSaveInstanceState一定会被执行
```

 总而言之，onSaveInstanceState的调用遵循一个重要原则，即当系统“未经你许可”时销毁了你的activity，则onSaveInstanceState会被系统调用，这是系统的责任，因为它必须要提供一个机会让你保存你的数据

### 二、activity的启动模式

四种启动模式分别是standard（标准模式）、singleTop(栈顶复用模式)、singleTask(栈内复用模式)、singleInstance（单实例模式 - 加强的singleTask模式）

##### standard

- 系统默认启动模式
- 不论存在与否，都会重新创建一个新的实例
- 多实例实现，谁启动了这个Activity、那么这个Activity就运行在启动它那个Activity所在栈

##### singleTop

- 判断需要启动的Activity是否为任务栈栈顶 ，如果是，则不会重新创建，如果不是，则会重新创建

- 不重新创建时候，该Activity的 onNewIntent(Intent intent) 方法会被回调，通过该方法的参数，可以取出当前请求的信息；

- 系统可能会杀死该Activity，杀死之后，启动情况与第一次启动相同，所以有必要在onCreate与onNewIntent方法中调用同一个处理数据的方法

  > 运用场景：常运用于通知栏弹出Notification，点击Notification跳转到指定的Activity，设置singleTop模式

##### singleTask

- 判断Activity所需任务栈内是否已经存在，如果存在，就把该Activity切换到栈顶（会导致在它之上的都会出栈）

- 如果所需任务栈都不存在，就会先创建任务栈再创建该Activity

- 可以理解为 顶置Activity+singleTop 的模式

  > 运用场景：可用来退出整个应用。主界面activity设为singleTas模式，要退出应用时转到主activity,从而将主activity之上的activity都清除，然后重写主activity的onNewIntent()方法，在里面加上finish(),即可退出所有activity。这种模式还适用于做浏览器、微博之类的应用

##### singleInstance

- 拥有singleTask的所有特性之外，此模式Activity只能单独地位于一个新的任务栈中

- 也就是，Activity启动之后，就会独自在一个新的任务栈中，下次肯定不会重新创建该Activity，除非被系统杀死

  > 运用场景：这种模式常运用于需要与程序分离的界面，如在SetupWizard（安装向导）中调用紧急呼叫就是适用这种模式

### 三、Intent和Intent-filter的匹配规则

##### 1、Intent和Intent Filter的介绍

**Intent**是抽象的数据结构，包含了一系列描述某个操作的数据，使得程序在运行时可以在程序中不同组件间通信或启动不同的应用程序。
可以通过startActivity(Intent)启动一个Activity,sendBroadcast(Intent))发送广播发送给感兴趣的BroadcastReceiver组件,startService(android.content.Intent))启动service，bindService()绑定服务。
**Intent Filter**顾名思义就是Intent的过滤器，组件通过定义Intent Filter可以决定哪些隐式Intent可以和该组件进行通讯
Intent分为隐式(implicit)Intent和显式(explicit)Intent。
**显式Intent通常用于在程序内部组件间通信**，已经明确的定义目标组件的信息，所以不需要系统决策哪个目标组件处理Intent，如下

```
Intent intent =new Intent(CRListDemo.this, GoogleMapDemo.class);
startActivity(intent);
```

其中CRListDemo和GoogleMapDemo都是用户自定义的组件，
**隐式Intent不指明目标组件的class，只定义希望的Action及Data等相关信息，由系统决定使用哪个目标组件。**如发送短信

##### 2.Intent和Intent-filter的匹配规则

Android系统通过对Intent和目标组件在AndroidManifest文件中定义的（也可以在程序中定义Intent Filter）进行匹配决定和哪个目标组件通讯。如果某组件未定义则只能通过显式的Intent进行通信。Intent的三个属性用于对目标组件选取的决策，分别是Action、Data（Uri和Data Type）、Category。匹配规则如下

- action匹配规则：要求intent中的action 存在 且 必须和过滤规则中的其中一个相同 区分大小写；
- category匹配规则：系统会默认加上一个android.intent.category.DEAFAULT，所以intent中可以不存在category，但如果存在就必须匹配其中一个；
- data匹配规则：data由两部分组成，mimeType和URI，要求和action相似。如果没有指定URI，URI但默认值为content和file（schema）

##### 3.利用Intent调用其他常见程序

a. 发送短信

```
Uri uri = Uri.parse("smsto:15800000000");
Intent i=newIntent(Intent.ACTION_SENDTO, uri);
i.putExtra("sms_body", "The SMS text");
startActivity(i);
```

b. 打电话

```
Uri dial = Uri.parse("tel:15800000000");
Intent i=newIntent(Intent.ACTION_DIAL, dial);
startActivity(i);
```

c. 发送邮件

```
Uri email = Uri.parse("mailto:abc@126.com;def@126.com");
Intent i=newIntent(Intent.ACTION_SENDTO, email);
startActivity(i);
```

d. 拍照

```
Intent i =newIntent(MediaStore.ACTION_IMAGE_CAPTURE);
String folderPath= Environment.getExternalStorageDirectory().getAbsolutePath() + File.separator + "AndroidDemo" +File.separator;
String filePath= folderPath + System.currentTimeMillis() + ".jpg";newFile(folderPath).mkdirs();
File camerFile=newFile(filePath);
i.putExtra(MediaStore.EXTRA_OUTPUT, Uri.fromFile(camerFile));
startActivityForResult(i,1);
```

e. 浏览网页

```
Uri web = Uri.parse("http://www.google.com");
Intent i=newIntent(Intent.ACTION_VIEW, web);
startActivity(i);
```

f. 查看联系人

```
Intent i =newIntent();
i.setAction(Intent.ACTION_GET_CONTENT);
i.setType("vnd.android.cursor.item/phone");
startActivityForResult(i,1);
```


