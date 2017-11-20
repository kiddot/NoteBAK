# Android应用程序的编译和打包过程

现在很多项目都开始编译和打包的自动化了。要想实现自动化需要我们对Android工程的编译和打包有一个深入的理解。



## 知识储备

|     名称     |             功能介绍             |
| :--------: | :--------------------------: |
|    aapt    |        Android资源打包工具         |
|    aidl    |   Android接口描述语言转化为java文件工具   |
|   javac    |        Java Compiler         |
|    dex     | 转化class文件为Davilk VM能识别的dex文件 |
| apkbuilder |            生成apk包            |
| jarsigner  |          jar文件的签名工具          |
|  zipalign  |           字节码对齐工具            |



## 1.打包资源文件，生成R.java文件

- **输入** ： res文件，assets文件（Android系统不对像res文件那样优化它），AndroidManifest.xml文件（包名在改文件读取，因为生成R.java文件需要包名），Android类库（jar文件）。其中只有那些类型为res/animator、res/anim、res/color、res/drawable（非Bitmap文件，即非.png、.9.png、.jpg、.gif文件）、res/layout、res/menu、res/values和res/xml的资源文件均会从文本格式的XML文件编译成二进制格式的XML文件



> XML资源文件之所要从文本格式编译成二进制格式
>
> - 二进制格式的XML文件占用空间更小。这是由于所有XML元素的标签、属性名称、属性值和内容所涉及到的字符串都会被统一收集到一个字符串资源池中去，并且会去重。有了这个字符串资源池，原来使用字符串的地方就会被替换成一个索引到字符串资源池的整数值，从而可以减少文件的大小。
>
>   ​
>
> - 二进制格式的XML文件解析速度更快。这是由于二进制格式的XML元素里面不再包含有字符串值，因此就避免了进行字符串解析，从而提高速度。



- **输出** ：打包好的资源（一般在Android工程的bin目录可以看到一个叫resources.ap_的文件就是它了）、R.java文件



- **工具 ** : aapt



## 2.处理AIDL文件，生成对应java文件

> 如果没有使用AIDL，这个过程将会省略



- **输入** : 源码文件，aidl文件，framework.aidl文件



- **输出** : java文件



- **工具** ： aidl工具



## 3.编译java文件，生成对应class文件



- **输入** ： 源码文件（包括R.java和AIDL生成的java文件），库文件（jar文件）



- **输出** ： class文件



- **工具** ： javac



## 4.class文件转化为Davilk VM支持的dex文件



- **输入** ： class文件（包括AIDL文件生成的class文件，R生成的class文件，源文件生成的class文件），库文件



- **输出** ： dex文件



- **工具** ： dx工具



## 5.打包生成未签名的apk文件



- **输入** ： 打包后的资源文件，打包后的类文件（dex文件），libs文件（包括so文件）



- **输出** ： 未签名的apk文件



- **工具** ： apkbuilder



## 6.对未签名的apk文件进行签名



- **输入** ： 未签名的apk文件



- **输出** ： 签名的apk文件



- **工具** ： jarsigner



## 7.对签名后的apk文件进行对齐处理



- **输入** ： 签名的apk文件



- **输出** ： 对齐猴的apk文件



- **工具** ： zipalign

