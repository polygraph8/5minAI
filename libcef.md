

# libcef 使用指南 

应用场景

嵌入一个兼容HTML5的浏览器控件到一个已经存在的本地应用。
创建一个轻量化的壳浏览器，用以托管主要用Web技术开发的应用。
有些应用有独立的绘制框架，使用CEF对Web内容做离线渲染。
使用CEF做自动化Web测试。

## 0、cef 中文使用指南  

https://github.com/fanfeilong/cefutil/blob/master/doc/CEF%20General%20Usage.md

- CefBrowser 是主要的浏览器host类，通过它的静态方法CefBrowser::CreateBrowser()方法创建新浏览器窗口.
- CefFrame 表示浏览器窗口里的一个框架（frame），每个浏览器窗口都有一个顶级的主框架，可通过CefBrowser::GetMainFrame()方法访问之.
- CefHandler 是传给CefBrowser::CreateBrowser()方法的最主要委托接口.
- CefRequest 表示请求数据，比如url, method, post data 和 headers.
- CefPostData 和 CefPostDataElement 表示可能是请求一部分的post数据.
- CefSchemeHandlerFactory 和 CefSchemeHandler 用于处理像myscheme://mydomain这样的自定义scheme.
- CefStreamReader, CefStreamWriter, CefReadHandler 和 CefWriteHandler 读写数据的简单的接口.
- CefV8Handler 和 CefV8Value 用于创建和访问Javascript对象.



##### C++ Wrapper

libcef 动态链接库导出 C API 使得使用者不用关心CEF运行库和基础代码。libcef_dll_wrapper 工程把 C API 封装成 C++ API同时包含在客户端应用程序工程中，与cefclient一样，源代码作为CEF二进制发布包的一部份共同发布。C/C++ API的转换层代码是由转换工具自动生成。UsingTheCAPI 页面描述了如何使用C API。

##### 进程

 CEF3是多进程架构的。"browser"被定义为主进程，负责窗口管理，界面绘制和网络交互。Blink的渲染和Js的执行被放在一个独立的"render" 进程中；  render进程还负责Js Binding和对Dom节点的访问。 默认的进程模型中，会为每个标签页创建一个新的"render"进程。其他进程按需创建，象管理插件的进程和处理合成加速的进程。 

主应用程序会被多次启动运行各自独立的进程。这是通过传递不同的命令行参数给CefExecuteProcess函数做到的。如果主应用程序很大，加载时间比较长，或者不能在非浏览器进程里使用，则宿主程序可使用独立的可执行文件去运行这些进程。这可以通过配置CefSettings.browser_subprocess_path变量做到。更多细节请参考`Application Structure`一节。

 CEF3的进程之间可以通过IPC进行通信。"Browser"和"Render"进程可以通过发送异步消息进行双向通信。甚至在Render进程可以注册在Browser进程响应的异步JavaScript API。更多细节，请参考`Inter-Process Communication`一节。 

##### 线程

在CEF3中，每个进程都会运行多个线程。完整的线程类型表请参照cef_thread_id_t。例如，在browser进程中包含如下主要的线程：

- **TID_UI** 线程是浏览器的主线程。如果应用程序在调用调用CefInitialize()时，传递CefSettings.multi_threaded_message_loop=false，这个线程也是应用程序的主线程。

- **TID_IO** 线程主要负责处理IPC消息以及网络通信。

- **TID_FILE** 线程负责与文件系统交互。

  

 由于CEF采用多线程架构，有必要使用锁和闭包来保证在多不同线程安全的传递数据。IMPLEMENT_LOCKING定义提供了Lock()和Unlock()方法以及AutoLock对象来保证不同代码块同步访问数据。CefPostTask函数组支持简易的线程间异步消息传递。 

 判断当前工作线程可以通过使用CefCurrentlyOn()方法，cefclient工程使用下面的定义来确保方法在期望的线程中被执行。 

```
#define REQUIRE_UI_THREAD()   ASSERT(CefCurrentlyOn(TID_UI));
#define REQUIRE_IO_THREAD()   ASSERT(CefCurrentlyOn(TID_IO));
#define REQUIRE_FILE_THREAD() ASSERT(CefCurrentlyOn(TID_FILE));
```

##### 引用计数

所有的框架类从CefBase继承，实例指针由CefRefPtr管理，CefRefPtr通过调用AddRef()和Release()方法自动管理引用计数。框架类的实现方式如下：

```
class MyClass : public CefBase {
 public:
 private:
 IMPLEMENT_REFCOUNTING(MyClass);  // Provides atomic refcounting implementation.
};
CefRefPtr<MyClass> my_class = new MyClass();
```

##### Strings 字符串

 CEF为字符串定义了自己的数据结构。下面是这样做的理由： 

- libcef包和宿主程序可能使用不同的运行时，对堆管理的方式也不同。所有的对象，包括字符串，需要确保和申请堆内存使用相同的运行时环境。

- libcef包可以编译为支持不同的字符串类型(UTF8，UTF16以及WIDE)。默认采用的是UTF16，默认字符集可以通过更改cef_string.h文件中的定义，然后重新编译来修改。当使用宽字节集的时候，切记字符的长度由当前使用的平台决定。

  ```
  typedef struct _cef_string_utf16_t {
    char16* str;  // Pointer to the string
    size_t length;  // String length
    void (*dtor)(char16* str);  // Destructor for freeing the string on the correct heap
  } cef_string_utf16_t;
  ```

```
std::string str = “Some UTF8 string”;

// Equivalent ways of assigning |str| to |cef_str|. Conversion from UTF8 will occur if necessary.
CefString cef_str(str);
cef_str = str;
cef_str.FromString(str);

// Equivalent ways of assigning |cef_str| to |str|. Conversion to UTF8 will occur if necessary.
str = cef_str;
str = cef_str.ToString();
```









##### 集成消息循环

 设置CefSettings.multi_threaded_message_loop=true（Windows平台下有效），这个设置项将导致CEF运行browser UI运行在单独的线程上，而不是在主线程上，这种场景下CefDoMessageLoopWork()或者CefRunMessageLoop()都不需要调用，CefInitialze()、CefShutdown()仍然在主线程中调用。你需要提供主程序线程通信的机制（查看cefclient_win.cpp中提供的消息窗口实例）。在Windows平台下，你可以通过命令行参数“--multi-threaded-message-loop”测试上述消息模型。 



##### CefSettings

- **single_process** 设置为true时，browser和renderer使用一个进程。此项也可以通过命令行参数“single-process”配置。查看本文中“进程”章节获取更多的信息。

- **browser_subprocess_path** 设置用于启动子进程单独执行器的路径。参考本文中“单独子进程执行器”章节获取更多的信息。

- **cache_path** 设置磁盘上用于存放缓存数据的位置。如果此项为空，某些功能将使用内存缓存，多数功能将使用临时的磁盘缓存。形如本地存储的HTML5数据库只能在设置了缓存路径才能跨session存储。

- **locale** 此设置项将传递给Blink。如果此项为空，将使用默认值“en-US”。在Linux平台下此项被忽略，使用环境变量中的值，解析的依次顺序为：LANGUAE，LC_ALL，LC_MESSAGES和LANG。此项也可以通过命令行参数“lang”配置。

- **log_file** 此项设置的文件夹和文件名将用于输出debug日志。如果此项为空，默认的日志文件名为debug.log，位于应用程序所在的目录。此项也可以通过命令参数“log-file”配置。

- **log_severity** 此项设置日志级别。只有此等级、或者比此等级高的日志的才会被记录。此项可以通过命令行参数“log-severity”配置，可以设置的值为“verbose”，“info”，“warning”，“error”，“error-report”，“disable”

- **resources_dir_path** 此项设置资源文件夹的位置。如果此项为空，Windows平台下cef.pak、Linux平台下devtools_resourcs.pak、Mac OS X下的app bundle Resources目录必须位于组件目录。此项也可以通过命令行参数“resource-dir-path”配置。

- **locales_dir_path** 此项设置locale文件夹位置。如果此项为空，locale文件夹必须位于组件目录，在Mac OS X平台下此项被忽略，pak文件从app bundle Resources目录。此项也可以通过命令行参数“locales-dir-path”配置。

- **remote_debugging_port** 此项可以设置1024-65535之间的值，用于在指定端口开启远程调试。例如，如果设置的值为8080，远程调试的URL为[http://localhost:8080。CEF或者Chrome浏览器能够调试CEF。此项也可以通过命令行参数“remote-debugging-port”配置。](http://localhost:8080。CEF或者Chrome浏览器能够调试CEF。此项也可以通过命令行参数"remote-debugging-port"配置。/)

  

##### CefBrowser and CefFrame /CefBrowserHost

 CefBrowser和CefFrame对象被用来发送命令给浏览器以及在回调函数里获取状态信息。每个CefBrowser对象包含一个主CefFrame对象，主CefFrame对象代表页面的顶层frame；同时每个CefBrowser对象可以包含零个或多个的CefFrame对象，分别代表不同的子Frame。例如，一个浏览器加载了两个iframe，则该CefBrowser对象拥有三个CefFrame对象（顶层frame和两个iframe）。 

```
browser->GetMainFrame()->LoadURL(some_url);  浏览器的主frame里加载一个URL
browser->GoBack();
class Visitor : public CefStringVisitor{ Visit(const CefString& string){}}
browser->GetMainFrame()->GetSource(new Visitor());
```

CefBrowser和CefFrame对象在Browser进程和Render进程都有对等的代理对象。在Browser进程里，Host（宿主）行为控制可以通过CefBrowser::GetHost()方法控制。例如，浏览器窗口的原生句柄可以用下面的代码获取：

```
// CefWindowHandle is defined as HWND on Windows, NSView* on Mac OS X
// and GtkWidget* on Linux.
CefWindowHandle window_handle = browser->GetHost()->GetWindowHandle();
```

CefBrowserHost

SendKeyEvent(const CefKeyEvent& event) ;

SendMouseClickEvent(const CefMouseEvent& event,     MouseButtonType type,      bool mouseUp,
                                   int clickCount))

  virtual void SendMouseMoveEvent(const CefMouseEvent& event,
                                  bool mouseLeave) = 0;

##### CefApp

- **OnBeforeCommandLineProcessing** 提供了以编程方式设置命令行参数的机会，更多细节，请参考`Command Line Arguments`一节。

- **OnRegisterCustomSchemes** 提供了注册自定义schemes的机会，更多细节，请参考`Request Handling`一节。

- **GetBrowserProcessHandler** 返回定制Browser进程的Handler，该Handler包括了诸如OnContextInitialized的回调。

- **GetRenderProcessHandler** 返回定制Render进程的Handler，该Handler包含了JavaScript相关的一些回调以及消息处理的回调。更多细节，请参考`JavascriptIntegration`和`Inter-Process Communication`两节。

  ```
  class MyApp : public CefApp,
                public CefBrowserProcessHandler,
                public CefRenderProcessHandler{
                   OnRenderProcessThreadCreated
                   OnContextInitialized
                   
                }
  ```

##### CefClient

CefClient提供访问browser-instance-specific的回调接口。单实例CefClient 开启数量的浏览器进程。以下为几个重要的回调：

​      比如处理browser的生命周期，右键菜单，对话框，通知显示， 拖曳事件，焦点事件，键盘事件等等。如果没有对某个特定的处理接口进行实现会造成什么影响，请查看cef_client.h文件中相关说明。 


```C
class MyHandler : public CefClient,
                  public CefContextMenuHandler,
                  public CefDisplayHandler,
                  public CefDownloadHandler,
                  public CefDragHandler,
                  public CefGeolocationHandler,
                  public CefKeyboardHandler,
                  public CefLifeSpanHandler,
                  public CefLoadHandler,
                  public CefRequestHandler {
                    virtual bool DoClose(CefRefPtr<CefBrowser> browser) OVERRIDE {
    // Allow or block browser window close...
  }
```

##### Browser Life Span

 Browser生命周期从执行 CefBrowserHost::CreateBrowser() 或者 CefBrowserHost::CreateBrowserSync() 开始。可以在CefBrowserProcessHandler::OnContextInitialized() 调用或者特殊平台例如windows的WM_CREATE 中方便的执行业务逻辑。 

 当browser对象创建后OnAfterCreated() 方法立即执行。宿主程序可以用这个方法来保持对browser对象的引用。 

```
OnAfterCreated{
    m_Browser = browser;
    m_BrowserId = browser->GetIdentifier();
}
```

 执行CefBrowserHost::CloseBrowser()销毁browser对象。 

```
browser->GetHost()->CloseBrowser(false);
```

```
case WM_CLOSE:
  if (g_handler.get() && !g_handler->IsClosing()) {
    CefRefPtr<CefBrowser> browser = g_handler->GetBrowser();
    if (browser.get()) {
      // Notify the browser window that we would like to close it. This will result in a call to 
      // MyHandler::DoClose() if the JavaScript 'onbeforeunload' event handler allows it.
      browser->GetHost()->CloseBrowser(false);

      // Cancel the close.
      return 0;
    }
  }
```

##### 离屏渲染(Off-Screen Rendering)

1、在离屏渲染模式下，CEF不会创建原生浏览器窗口。CEF为宿主程序提供无效的区域和像素缓存区，而宿主程序负责通知鼠标键盘以及焦点事件给CEF。离屏渲染目前不支持混合加速，所以性能上可能无法和非离屏渲染相比。离屏浏览器将收到和窗口浏览器同样的事件通知，

下面介绍如何使用离屏渲染： 

- 实现CefRenderHandler接口。除非特别说明，所有的方法都需要覆写。

- 调用CefWindowInfo::SetAsOffScreen()，将CefWindowInfo传递给CefBrowserHost::CreateBrowser()之前还可以选择设置CefWindowInfo::SetTransparentPainting()。如果没有父窗口被传递给SetAsOffScreen,则有些类似上下文菜单这样的功能将不可用。

- CefRenderHandler::GetViewRect方法将被调用以获得所需要的可视区域。

- CefRenderHandler::OnPaint() 方法将被调用以提供无效区域（脏区域）以及更新过的像素缓存。cefclient程序里使用OpenGL绘制缓存，但你可以使用任何别的绘制技术。

- 可以调用CefBrowserHost::WasResized()方法改变浏览器大小。这将导致对GetViewRect()方法的调用，以获取新的浏览器大小，然后调用OnPaint()重新绘制。

- 调用CefBrowserHost::SendXXX()方法通知浏览器的鼠标、键盘和焦点事件。

- 调用CefBrowserHost::CloseBrowser()销毁浏览器。

  

##### 投递任务(Posting Tasks)

 任务(Task)可以通过CefPostTask在一个进程内的不同的线程之间投递。CefPostTask有一系列的重载方 

    CefPostTask(TID_UI, base::Bind(&SimpleHandler::CloseAllBrowsers, this,
                                   force_close));

```
CefPostTask(TID_IO, NewCefRunnableFunction(MyFunction, param1, param2));
```

##### 进程间通信(Inter-Process Communication (IPC))

 由于CEF3运行在多进程环境下，所以需要提供一个进程间通信机制。CefBrowser和CefFrame对象在Borwser和Render进程里都有代理对象。CefBrowser和CefFrame对象都有一个唯一ID值绑定，便于在两个进程间定位匹配的代理对象。 

 CefFrame::GerIdentifier()获取CefFrame的ID，并通过进程间消息发送给另一个进程，然后在接收端通过CefBrowser::GetFrame()找到对应的CefFrame。 

```
// Helper macros for splitting and combining the int64 frame ID value.
#define MAKE_INT64(int_low, int_high) \
    ((int64) (((int) (int_low)) | ((int64) ((int) (int_high))) << 32))
#define LOW_INT(int64_val) ((int) (int64_val))
#define HIGH_INT(int64_val) ((int) (((int64) (int64_val) >> 32) & 0xFFFFFFFFL))

// Sending the frame ID.
const int64 frame_id = frame->GetIdentifier();
args->SetInt(0, LOW_INT(frame_id));
args->SetInt(1, HIGH_INT(frame_id));

// Receiving the frame ID.
const int64 frame_id = MAKE_INT64(args->GetInt(0), args->GetInt(1));
CefRefPtr<CefFrame> frame = browser->GetFrame(frame_id);
```



##### 异步JavaScript绑定

 JavaScritp被集成在Render进程，但是需要频繁和Browser进程交互。 JavaScript API应该被设计成可使用闭包异步执行。 

#####  通用消息转发 

 从1574版本开始，CEF提供了在Render进程执行的JavaScript和在Browser进程执行的C++代码之间同步通信的转发器。应用程序通过C++回调函数（OnBeforeBrowse, OnProcessMessageRecieved, OnContextCreated等）传递数据。Render进程支持通用的JavaScript回调函数注册机制，Browser进程则支持应用程序注册特定的Handler进行处理。 

 A. The query is canceled in JavaScript using the |window.cefQueryCancel|
//    function.
// B. The query is canceled in C++ code using the Callback::Failure function.
// C. The context associated with the query is released due to browser
//    destruction, navigation or renderer process termination.

 完整的用法请参考wrapper/cef_message_router.h 



#####  自定义实现 

 一个CEF应用程序也可以提供自己的异步JavaScript绑定。典型的实现如下： 

```
// In JavaScript register the callback function.
app.setMessageCallback('binding_test', function(name, args) {
  document.getElementById('result').value = "Response: "+args[0];
});
```

```
// Map of message callbacks.
typedef std::map<std::pair<std::string, int>,
                 std::pair<CefRefPtr<CefV8Context>, CefRefPtr<CefV8Value> > >
                 CallbackMap;
CallbackMap callback_map_;

// In the CefV8Handler::Execute implementation for “setMessageCallback”.
if (arguments.size() == 2 && arguments[0]->IsString() &&
    arguments[1]->IsFunction()) {
  std::string message_name = arguments[0]->GetStringValue();
  CefRefPtr<CefV8Context> context = CefV8Context::GetCurrentContext();
  int browser_id = context->GetBrowser()->GetIdentifier();
  callback_map_.insert(
      std::make_pair(std::make_pair(message_name, browser_id),
                     std::make_pair(context, arguments[1])));
}
```

1. Render进程发送异步进程间通信到Browser进程。
2.  Browser进程接收到进程间消息，并处理。 
3.  Browser进程处理完毕后，发送一个异步进程间消息给Render进程，返回结果。 
4.  Render进程接收到进程间消息，则调用最开始保存的JavaScript注册的回调函数处理之。 
5. 在CefRenderProcessHandler::OnContextReleased()里释放JavaScript注册的回调函数以及其他V8资源。

##### 同步请求

 Browser进程可以通过自定义scheme Handler或者网络交互处理XMLHttpRequest。 

```
browser->GetMainFrame()->LoadURL(some_url);
```

```
// Create a CefRequest object.
CefRefPtr<CefRequest> request = CefRequest::Create();
request->SetURL(some_url);
request->SetMethod(“POST”);
request->SetPostData(postData);
CefFrame::LoadRequest(request)
```

 CefURLRequest 

```
    CefURLRequest::Status status = request->GetRequestStatus();
    CefURLRequest::ErrorCode error_code = request->GetRequestError();
    CefRefPtr<CefResponse> response = request->GetResponse();
    
    CefRefPtr<CefURLRequest> url_request = CefURLRequest::Create(request, client.get());
```



##### Scheme响应(Scheme Handler)

```
CefRegisterSchemeHandlerFactory("client", “myapp”, new MySchemeHandlerFactory());
```

 scheme Handler类可以被用在内置shcme(HTTP,HTTPS等），也可以被用在自定义scheme上。当使用内置shceme，选择一个对你的应用程序来说唯一的域名。实现CefSchemeHandlerFactory和CefResoureHandler类去处理请求并返回响应数据。可以参考cefclient/sheme_test.h的例子 

##### 请求拦截(Request Interception)

 CefRequestHandler::GetResourceHandler()方法支持拦截任意请求。参考client_handler.cpp。 

```CefRefPtr&lt;CefResourceHandler&gt; MyHandler::GetResourceHandler(
CefRefPtr<CefResourceHandler> MyHandler::GetResourceHandler(
      CefRefPtr<CefBrowser> browser,
      CefRefPtr<CefFrame> frame,
      CefRefPtr<CefRequest> request) {
  if (...)
    return new MyResourceHandler();
  return NULL;
}
```

##### websocket 

cef_server.h   

CefServerHandler::OnWebSocketConnected



## 1 下载和建立项目

### 1 下载

http://opensource.spotify.com/cefbuilds/index.html



###  2 cmake-gui 生成vs 项目

![1603958826705](libcef/1603958826705.png)



![1603958933044](libcef/1603958933044.png)

cefsimple :  libcef_dll_wrapper.lib , libcef.lib 



### 3  建立项目过程

1、 首先就是建一个空的win32项目，例如名字为TestLibCef 

2、 cefsimple目录（注意是拷贝文件夹）拷贝到新工程下并包含在项目中 

3、 并在TestLibCef\TestLibCef文件夹下，新建一个dll文件夹，Debug目录下的文件全部拷贝到该文件夹下

4、 把resource目录下的文件全部拷贝到该文件夹下（**TestLibCef\TestLibCef\dll**）

5、把include文件夹拷贝到该文件夹下（注意是拷贝文件夹）（**TestLibCef\TestLibCef\dll**）

6、libcef_dll_wrapper.lib文件拷贝到该文件夹下（**TestLibCef\TestLibCef\dll**）
(如果你要发布你的应用程序了，那么你就应该拷贝相应的release目录下的文件) 

7、在工程中添加一些头文件和源文件

8、 TestLibCef属性，附加包含目录  **TestLibCef\TestLibCef\dll**  +TestLibCef\TestLibCef

9、 C/C++下代码生成中，运行库改为“**多线程调试MTD**” ,预编译头：不使用预编译头	

10、 链接器，常规，附加库目录为：   C:\Program Files (x86)\Windows Kits\10\Lib\10.0.1015**0.0\ucrt\x86** 

11、 链接器，输入， 

 D:\test\TestLibCef\TestLibCef\dll\libcef_dll_wrapper.lib 

D:\test\TestLibCef\TestLibCef\dll\libcef.lib 

D:\test\TestLibCef\TestLibCef\dll\cef_sanbox.lib 

12、链接器 -高级-目标计算机   

13、 关闭VS2015；
打开VS2015软件（不点击任何解决方案）；
选择 **文件** ->**打开** ->**项目**， 找到之前建立的TestLibCef的sln文件。 

## 2 SimpleApp 简单例子



![1605077137772](libcef/1605077137772.png)







CefEnableHighDPISupport();   

void* sandbox_info = NULL;   

 main_args(hInstance);   # 命令行参数







##### **1 创建应用类**

CefRefPtr<SimpleApp> **app**(new SimpleApp);    *// 创建CefRef实例，SimpleApp implements application-level callbacks* 

​		CefSettings settings;   **settings**.no_sandbox = true;  // 设置没有沙盒*

​		CefWindowInfo **window_info**;  window_info.SetAsPopup(NULL, L"百度"); window_info.SetTransparentPainting(FALSE);    //设置window 窗口*

​		CefBrowserSettings **browser_settings**:: DefaultEncoding,UserStyleSheetLocation*
*RemoteFonts,JavaScript,JavaScriptOpenWindows* 

##### **2 初始化CEF** 

**CefInitialize**(main_args, settings, **app.get(),** sandbox_info);   // 

​                    初始化完之后调用simpleApp的OnContextInitialized 

                     ```C++
  // SimpleHandler implements browser-level callbacks. 实现浏览器级别的回调
  CefRefPtr<SimpleHandler> handler(new SimpleHandler(use_views)); 例如onBeforeClose

  // Specify CEF browser settings here.   设置browser 
  CefBrowserSettings browser_settings;
 有视图
      //创建浏览器视图.
    CefRefPtr<CefBrowserView> **BROWSER_VIEW** = CefBrowserView::CreateBrowserView(
        handler, url, browser_settings, nullptr, nullptr,
        new SimpleBrowserViewDelegate());
    // 创建窗口 Create the Window. It will show itself after creation.
    CefWindow::CreateTopLevelWindow(new SimpleWindowDelegate(  **BROWSER_VIEW** ));
 无视图
    CefWindowInfo window_info;
    window_info.SetAsPopup(NULL, "cefsimple");
    CefBrowserHost::CreateBrowser(window_info, handler, url, browser_settings,
                                  nullptr, nullptr);

                     ```

##### 3 消息循环关闭

CefRunMessageLoop();   *// Shut down CEF.* 

CefShutdown(); 



##### 4  SimpleHandler 说明

ExampleCefHandler 继承的handler 如下:

​          CefClient,   CefContextMenuHandler,   CefDisplayHandler, cefDownloadHandler,  CefDragHandler,    CefGeolocationHandler,  CefKeyboardHandler,                          public CefLifeSpanHandler,   CefLoadHandler,CefRequestHandler

实现的接口：

```c++
void ExampleCefHandler::OnAfterCreated(CefRefPtr<CefBrowser> browser) {
  REQUIRE_UI_THREAD();
  AutoLock lock_scope (this);

  this->browser = browser;

  CefLifeSpanHandler::OnAfterCreated (browser);
}

CefRefPtr<CefBrowser> ExampleCefHandler::GetBrowser () {   return browser; } 
修改 关闭前
void ExampleCefHandler::OnBeforeClose(CefRefPtr<CefBrowser> browser) 
 {  
     REQUIRE_UI_THREAD();  
     AutoLock lock_scope (this);   browser = NULL;  
     PostMessage(application_message_window_handle, WM_COMMAND, QUIT_CEF_EXAMPLE, 0);   
     CefLifeSpanHandler::OnBeforeClose (browser);
 } 
修改browser title
OnTitleChange(CefRefPtr<CefBrowser> browser,const CefString& title) {
  CEF_REQUIRE_UI_THREAD();
  if (use_views_) {
    // Set the title of the window using the Views framework.
    CefRefPtr<CefBrowserView> browser_view = CefBrowserView::GetForBrowser(browser);
    if (browser_view) {
      CefRefPtr<CefWindow> window = browser_view->GetWindow();
      if (window)
        window->SetTitle(title);
    }
  } else {
    // Set the title of the window using platform APIs.
    PlatformTitleChange(browser, title);
  }
}
void OnAfterCreated(CefRefPtr<CefBrowser> browser) {browser_list_.push_back(browser);}

void SimpleHandler::OnBeforeClose(CefRefPtr<CefBrowser> browser) {
  // Remove from the list of existing browsers.
  BrowserList::iterator bit = browser_list_.begin();
  for (; bit != browser_list_.end(); ++bit) {
    if ((*bit)->IsSame(browser)) {
      browser_list_.erase(bit);
      break;
    }
  }
  if (browser_list_.empty()) CefQuitMessageLoop();
}

```



## 3 CefClient 例程





## 4 另外的例子

### 4.1自建窗口

https://blog.csdn.net/wangshubo1989/article/details/50199599

**WinMain**

   CefInitialize (main_args, settings, app.get()); 
   **CreateBrowserWindow** (hInstance, nCmdShow);      
   application_message_window_handle = **CreateMessageWindow** (hInstance); 

**CreateBrowserWindow** {

​        WNDCLASSEX wcex = { 0 }; 

​       wcex.lpfnWndProc   = **BrowserWindowWndProc**;
​      RegisterClassEx (&wcex);   

​      HWND window_handle (CreateWindow (BROWSER_WINDOW_CLASS, BROWSER_WINDOW_CLASS,    WS_OVERLAPPEDWINDOW | WS_CLIPCHILDREN, CW_USEDEFAULT, 0,  

}



 **BrowserWindowWndProc**  

 case WM_CREATE:         {            

​            example_cef_handler = new ExampleCefHandler(); 

​             RECT rect = { 0 };            

​             GetClientRect (window_handle, &rect);             

​             CefWindowInfo info;            

​            info.SetAsChild(window_handle, rect);             

​            CefBrowserSettings settings;            

​             CefBrowserHost::**CreateBrowser**(info, example_cef_handler.get(),             CefString ("http://www.google.com"), settings, NULL); 



 **MessageWindowWndProc** (HWND window_handle, UINT message, WPARAM w_param, LPARAM l_param) {   LRESULT result (0);   

switch (message)   {      

​	case WM_COMMAND:         {            

​                 if (w_param == QUIT_CEF_EXAMPLE)            

​                 {               

​                             PostQuitMessage(0); 



### 4.2另外一个例子  MFC 

https://blog.csdn.net/blackwoodcliff/article/details/74276848

十分详细





https://blog.csdn.net/gaga392464782/article/details/49490101

CefMFCApp::InitInstance()

```
CefRefPtr<clientApp> app(new clientApp);
CefExecuteProcess(main_args,app.get(),NULL);
setting.multi_threaded_message_loop = true ;
**CefInitialize**(main_args, settings, **app.get(),** sandbox_info); 
```

CefMFCApp::ExitInstance()

​       CefShutdown()

Dlg::OnCreate():

    CefWindowInfo window_info;
    window_info.SetAsPopup(NULL, "cefsimple");
    CefBrowserHost::CreateBrowser(window_info, handler, url, browser_settings,nullptr, nullptr);





## 5 CEF 基本功能


### 创建浏览器-浏览url 

std::wstring url = L"www.baidu.com";
CefBrowserHost::CreateBrowser(info, **g_web_browser_client**.get(), CefString(url), browserSettings, NULL);

std::wstring **url2** = "www.google.com"
CefRefPtr<CefBrowser> browser = **g_web_browser_client**->GetBrowser();
browser->GetMainFrame()->LoadURL(CefString(**url2**)); 



### 设置cookie

std::wstring **username_key** = L"username"*;*    

std::wstring **domain** = L"blog.csdn.net"     



CefRefPtr<CefCookieManager> **manager** = CefCookieManager::GetGlobalManager()*;*    

CefCookie cookie*;*    

CefString(&cookie.name).**FromWString**(**username_key**.c_str())*;*    CefString(&cookie.domain).FromWString(**domain**.c_str())*;*    

CefString(&cookie.path).FromASCII("/")*;*    



**cookie**.has_expires = false*;*     

**domain** = L"https://" + domain*;*    

CefPostTask(TID_IO, NewCefRunnableMethod(**manager**.get(),&CefCookieManager::SetCookie,CefString(domain.c_str()), **cookie**))*;*
 //创建浏览器    

CefBrowserHost::CreateBrowser(info, g_web_browser_client.get(), domain.c_str(), browserSettings, NULL)*;* 



### C++调用js

CefRefPtr<CefFrame> **frame** = browser->GetMainFrame(); 

frame->ExecuteJavaScript("alert('ExecuteJavaScript works!');",  frame->GetURL(), 0);

在本人的依赖中有 GetCefInstance()->RunJS(pTestWeb->GetWebID(), L"sendMessage", 1, JsDataStr.GetBuffer());

 第一个参数为该网页的ID号，第二个为js的函数名，第三个为总的参数个数，第四个为参数，详情请看源代码。



https://blog.csdn.net/mfcing/article/details/44539035?utm_source=blogxgwz6 



### JS调用C++函数

![1604464620749](libcef/1604464620749.png)





重写 CefRenderProcessHandler的OnContextCreated接口 

{

创建函数对象 并将JS函数绑定到C++函数指针上面的过程 

CefRefPtr<CefV8Handler> myV8handle = new CCefV8Handler();

CefRefPtr<CefV8Value> myFun = CefV8Value::CreateFunction(L"SetAppState", myV8handle);

static_cast<CCefV8Handler*>(myV8handle.get())->AddFun(L"SetAppState", &CChromeJsCallback::JsSetAppState);

​		pObjApp->SetValue(L"SetAppState", myFun, V8_PROPERTY_ATTRIBUTE_NONE);



## 6 其他

https://blog.csdn.net/cloudmq/article/details/80598590  基于cef3浏览器开发经历



### 框架接口说明

CefBrowser是主要的浏览器窗口类，可以用静态的函数CreateBrowser() 和CreateBrowserSync() 来创建一个新的浏览器窗口。
CefFrame 代表一个浏览器窗口的框架，每个浏览器窗口有一个顶层的主框架，而这个主框架可以用GetMainFrame() 方法得到。
CefClient是主浏览器窗口的代表接口，这个接口做为参数传递给CreateBrowser()
CefRequest 代表URL，方法，发送数据和头文件等这样的请求。
CefSchemeHandleFactory 类是被用来处理像myscheme://mydomain类似客户计划的请求
CefReadHandler和CefWriteHandle是一个读写数据的简单接口。
CefV8Handler,CefV8Value和CefV8Context是被用来创建和访问JavaScript对象。 

### CefBrowser和CefFrame 方法

 Back, Forward, Reload and Stop Load。控件浏览器的导航
Undo, Redo, Cut, Copy, Paste, Delete, Select All.控件目标框架的选取
Print。打印目标框架
Get Source。以字符串的形式来获取目标框架的HTML源码
View Source. 用缓存文件来保存目的框架的HTML源码，并且用系统默认的文本查看器打开
Load URL.加载特定的URL到目标框架
Load String. 加载一个特定的字符串到目标框架，通过一个随意指定的虚拟URL
Load Stream. 加载一个特定的二进制文件到目标框架，通过一个随意指定的虚拟URL
Load Request, 加载一个特定的请求到目标框架
Execute JavaScript: 在目标框架里面执行一个特定的Javscript命令
Zoom。 缩放特定框架的网页内容 



### linux 错误处理

```
  XSetErrorHandler(XErrorHandlerImpl);
  XSetIOErrorHandler(XIOErrorHandlerImpl);
  
int XErrorHandlerImpl(Display *display, XErrorEvent *event) {
  LOG(WARNING)
        << "X error received: "
        << "type " << event->type << ", "
        << "serial " << event->serial << ", "
        << "error_code " << static_cast<int>(event->error_code) << ", "
        << "request_code " << static_cast<int>(event->request_code) << ", "
        << "minor_code " << static_cast<int>(event->minor_code);
  return 0;
}
```





调用例子

https://blog.csdn.net/gaga392464782/article/details/49490101

https://dabaojian.blog.csdn.net/article/details/50189577?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.channel_param



## 7 自动化测试

7.1 webdriver 原理



```java
nameToUrl = ImmutableMap.<String, CommandInfo>builder()        .put(NEW_SESSION, post("/session"))        .put(QUIT, delete("/session/:sessionId"))        .put(GET_CURRENT_WINDOW_HANDLE, get("/session/:sessionId/window_handle"))        .put(GET_WINDOW_HANDLES, get("/session/:sessionId/window_handles"))        .put(GET, post("/session/:sessionId/url"))             // The Alert API is still experimental and should not be used.        .put(GET_ALERT, get("/session/:sessionId/alert"))        .put(DISMISS_ALERT, post("/session/:sessionId/dismiss_alert"))        .put(ACCEPT_ALERT, post("/session/:sessionId/accept_alert"))        .put(GET_ALERT_TEXT, get("/session/:sessionId/alert_text"))        .put(SET_ALERT_VALUE, post("/session/:sessionId/alert_text"))
```