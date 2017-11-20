# WebView总结

## WebView的基本使用

- 权限

```java
 <uses-permission android:name="android.permission.INTERNET" />
<!--另外附上一些可能会用到的权限：-->
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<!--获取网络状态权限（情景：WebView联网前，应检查当前网络状态）-->
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION"/>
<!--通过WiFi或移动基站的方式获取用户错略的经纬度信息-->
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<!--通过GPS芯片接收卫星的定位信息权限（情景：结合HTML5使用Geolocation API获取位置时）-->
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
<!--读写存储权限（情景：拍照/选择图片，涉及图片读取、编辑、压缩等功能时）-->
```



- 常用配置

```java
 WebSettings  mWebSetting = mWebView.getSettings();  //获取WebSetting
 setJavaScriptEnabled(true);//让WebView支持JavaScript
 setDomStorageEnabled(true);//启用H5 DOM API （默认false）
 setDatabaseEnabled(true);//启用数据库api（默认false）可结合 setDatabasePath 设置路径
 setCacheMode(WebSettings.LOAD_DEFAULT)//设置缓存模式
 setAppCacheEnabled(true);//启用应用缓存（默认false）可结合 setAppCachePath 设置缓存路径
 setAppCacheMaxSize()//已过时，高版本API上，系统会自行分配
 setPluginsEnabled(true);  //设置插件支持
 setRenderPriority(RenderPriority.HIGH);  //提高渲染的优先级
 setUseWideViewPort(true);  //将图片调整到适合webview的大小
 setLoadWithOverviewMode(true); // 缩放至屏幕的大小
 setSupportZoom(true);  //支持缩放，默认为true
 setBuiltInZoomControls(true); //设置内置的缩放控件（若SupportZoom为false，该设置项无效）
 setDisplayZoomControls(false); //隐藏原生的缩放控件
 setLayoutAlgorithm(LayoutAlgorithm.SINGLE_COLUMN); //支持内容重新布局
 supportMultipleWindows();  //支持多窗口
 setCacheMode(WebSettings.LOAD_CACHE_ELSE_NETWORK);  //关闭webview中缓存
 setAllowFileAccess(true);  //设置可以访问文件
 setNeedInitialFocus(true); //当webview调用requestFocus时为webview设置节点
 setJavaScriptCanOpenWindowsAutomatically(true); //支持通过JS打开新窗口
 setLoadsImagesAutomatically(true);  //自动加载图片
 setDefaultTextEncodingName("utf-8");//设置编码格式
```



- 加载页面的三种方式

```java
//网页
private static final String URL_NET = "http://www.google.com"; // 记得加 "http://"
//assets 中的 html 资源
private static final String URL_LOCAL ="file:///android_asset/xxx.html路径"; 
//SD 卡中的 html 资源
private static final String URL_SD_CARD ="content://com.android.htmlfileprovider/mnt/sdcard/xxx.html"; 
mWebView.loadUrl(URL_NET);
mWebView.loadUrl(URL_LOCAL);
mWebView.loadUrl(URL_SD_CARD);
```



## WebViewClient & WebChromeClient

### WebViewClient主要用于帮助webview处理各种通知和请求时间

- shouldOverrideUrlLoading 加载时调用，可捕获url
- onPageStart 开始加载时调用
- onPageFinish 加载完成时调用
- onReceiveError 接收到错误信息时调用，通常用来处理404等加载错误， **但是在低于API23时，该方法失效，可能是大于API23不再接受网页连接错误，而是WebView自身的错误**



### WebChromeClient用于辅助WebView处理JS的对话框，网站图标，位置，加载进度等信息

- onProgressChanged 加载进度
- onReceivedTitle、onReceivedIcon 获取网页标题和图标



## WebView与JS的交互

> 前提条件：setJavaScriptEnabled(true)



- 调用JS方法

  **mWebView.loadUrl("javascript: 方法名('"+参数+"')");**

- js调用android方法

  ```java
   //使用方法
   mWebView.addJavascriptInterface(MethodObject,"name");
   //还需要写一个方法类
   class MethodObject extends Object {
      //无参函数，js中通过：var str = window.name.HtmlcallJava(); 获取到
      @JavascriptInterface
      public String HtmlcallJava() {
          return "Html call Java";
      }
      //有参函数，js中通过：window.jsObj.HtmlcallJava2("IT-homer blog");
      @JavascriptInterface
      public String HtmlcallJava2(final String param) {
          return "Html call Java : " + param;
      }
  }
  ```



## WebView优化

### 1.页面加载速度

> 影响页面加载速度的因素有非常多，每次加载的过程中都会有较多的网络请求，除了 web 页面自身的 URL 请求，还会有 web 页面外部引用的JS、CSS、字体、图片等等都是个独立的 http 请求。这些请求都是串行的，这些请求加上浏览器的解析、渲染时间就会导致 WebView 整体加载时间变长



- #### 选择合适的 WebView 缓存（二次加载速度优化）

  - **WebView缓存**分为：页面缓存和数据缓存。**页面缓存**指加载网页时，对页面或资源数据的缓存（一般使用RE管理器进入目录： “/data/data/(packageName)/cache/org.chromium.android_webview“可看 ）。**数据缓存**又分为 AppCache 与 DOM Storage 。AppCache可以有选择地缓存我们所想要缓存的东西；DOM Storage 则是HTML5的一个缓存机制，常用于存储简单的表单数据

  - WebView的缓存模式

    |     LOAD_CACHE_ONLY     |             不使用网络，只读取本地缓存数据              |
    | :---------------------: | :--------------------------------------: |
    |      LOAD_DEFAULT       |        根据cache-control决定是否从网络上取数据        |
    |    LOAD_CACHE_NORMAL    | API level 17中已经废弃, 从API level 11开始作用同LOAD_DEFAULT模式 |
    |      LOAD_NO_CACHE      |              不使用缓存，只从网络获取数据              |
    | LOAD_CACHE_ELSE_NETWORK |    只要本地有，无论是否过期，或者no-cache，都使用缓存中的数据     |

    ​

- 缓存的数据转移到SD卡

  重写Context 类中的getCacheDir方法即可

  ```java
   @Override
      public void onCreate() {
          super.onCreate();

          if (Environment.MEDIA_MOUNTED.equals(Environment.getExternalStorageState())){
              File externalStorageDir = Environment.getExternalStorageDirectory();
              if (externalStorageDir != null){
                  // {SD_PATH}/Android/data/
                  extStorageAppBasePath = new File(externalStorageDir.getAbsolutePath() +
                          File.separator +"Android" + File.separator + "data"+File.separator + getPackageName());
                  Log.i(TAG,extStorageAppBasePath+"=====");

              }

              if (extStorageAppBasePath != null){
                  extStorageAppCachePath = new File(extStorageAppBasePath.getAbsolutePath()+File.separator + "webViewCache");
                  Log.d("extStorageAppCachePath","extStorageAppCachePath = "+extStorageAppCachePath);
                  boolean isCachePathAvailable = true;
                  if (!extStorageAppCachePath.exists()){
                      isCachePathAvailable = extStorageAppCachePath.mkdirs();
                      if (!isCachePathAvailable){
                          extStorageAppCachePath = null;
                      }
                  }
              }
          }
      }
  ```



      @Override
      public File getCacheDir() {
    
          if (extStorageAppCachePath != null){
              return extStorageAppCachePath;
          }else{
              return super.getCacheDir();
          }
      }
  ```

  ​



- **资源预加载**（首次速度优化）

> 缓存技术，能优化二次启动 WebView 的加载速度，那首次加载 H5 页面的速度该怎么优化?一种思路就是外部依赖的 JS、CSS、图片等资源先提前在wifi环境下先预载好




WebView mWebView = (WebView) findViewById(R.id.webview);
mWebView.setWebViewClient(new WebViewClient() {
            @Override
            public WebResourceResponse shouldInterceptRequest(WebView webView, final String url) {
                WebResourceResponse response = null;
              	// 检查该资源是否已经提前下载完成。我采用的策略是在应用启动时，用户在 wifi 的网络环境下				// 提前下载 H5 页面需要的资源。
                boolean resDown = JSHelper.isURLDownValid(url);
                if (resDown) {
                    jsStr = JsjjJSHelper.getResInputStream(url);
                    if (url.endsWith(".png")) {
                        response = getWebResourceResponse(url, "image/png", ".png");
                    } else if (url.endsWith(".gif")) {
                        response = getWebResourceResponse(url, "image/gif", ".gif");
                    } else if (url.endsWith(".jpg")) {
                        response = getWebResourceResponse(url, "image/jepg", ".jpg");
                    } else if (url.endsWith(".jepg")) {
                        response = getWebResourceResponse(url, "image/jepg", ".jepg");
                    } else if (url.endsWith(".js") && jsStr != null) {
                        response = getWebResourceResponse("text/javascript", "UTF-8", ".js");
                    } else if (url.endsWith(".css") && jsStr != null) {
                        response = getWebResourceResponse("text/css", "UTF-8", ".css");
                    } else if (url.endsWith(".html") && jsStr != null) {
                        response = getWebResourceResponse("text/html", "UTF-8", ".html");
                    }
                }
                // 若 response 返回为 null , WebView 会自行请求网络加载资源。 
                return response;
            }
        });

private WebResourceResponse getWebResourceResponse(String url, String mime, String style) {
        WebResourceResponse response = null;
        try {
            response = new WebResourceResponse(mime, "UTF-8", new FileInputStream(new File(getJSPath() + TPMD5.md5String(url) + style)));
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        }
        return response;
    }

public String getJsjjJSPath() {
        String splashTargetPath = JarEnv.sApplicationContext.getFilesDir().getPath() + "/JS";
        if (!TPFileSysUtil.isDirFileExist(splashTargetPath)) {
            TPFileSysUtil.createDir(splashTargetPath);
        }
        return splashTargetPath + "/";
    }
  ```



- JS 脚本本地化（首次加载优化）

  > ​比预加载更粗暴的优化方法是直接将常用的 JS 脚本本地化，直接打包放入 apk 中。比如 H5 页面获取用户信息，设置标题等通用方法，就可以直接写入一个 JS 文件，放入 asserts 文件夹，在 WebView 调用了onPageFinished() 方法后进行加载。但是，**该 JS 文件中需要写入一个 JS 文件载入完毕的事件**



- 使用第三方WebView内核

> Android 4.4 版本 Google 使用了Chromium 替代 Webkit 作为 WebView 内核,**但是我们可以使用腾讯集成的SDK，就可以使用X5内核了**，更换第三方内核速度之后各方面速度都会有较大的提升





### 2.内存泄露&&内存优化

> ​	WebView解析网页时会申请Native堆内存用于保存页面元素，当页面较复杂时会有很大的内存占用。为了打开新页面时，为了能快速回退，之前页面占用的内存也不会释放。有时浏览十几个网页，都会占用几百兆的内存。这样加载网页较多时，会导致系统不堪重负，最终强制关闭应用，也就是出现应用闪退或重启。



> ​	由于占用的都是Native堆内存，所以实际占用的内存大小不会显示在常用的DDMS Heap工具中（这里看到的只是Java虚拟机分配的内存，一般即使Native堆内存已经占用了几百兆，这里显示的还只是几兆或十几兆）。只有使用adb shell中的一些命令比如dumpsys meminfo 包名，或者在程序中使用Debug.getNativeHeapSize()才能看到。



> ​	由于WebView的一个BUG，即使它所在的Activity(或者Service)结束也就是onDestroy()之后，或者直接调用WebView.destroy()之后，它所占用这些内存也不会被释放。



**解决这个问题最直接的方法是：把使用了WebView的Activity(或者Service)放在单独的进程里。然后在检测到应用占用内存过大有可能被系统干掉或者它所在的Activity(或者Service)结束后，调用System.exit(0)，主动Kill掉进程。由于系统的内存分配是以进程为准的，进程关闭后，系统会自动回收所有内存。**