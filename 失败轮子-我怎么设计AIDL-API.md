# 单一入口api

类似于后台的接口设计, 分为3层

```
client -> api service -> router -> controller 
```

1. api: 实现 aidl stub 的 service。跨进程的调用入口，将请求重定向到本进程的 router
2. router: 根据请求参数分发到各自的 controller 
3. controller: 处理请求


为了扩展性，aidl 提供的接口是通用的，必须可以接收后期各种新的功能。入口不能是 `doA(xx), doB(xx), doC(xx)` 这种具体的形式，

而是 `do(request)`。为了使用方便，可以在client加一层 **ApiSdk** 负责 `doXX(xx) -> do(request)`。

之后更改功能，只需要改对应的Controller实现。

这个设计的优点是，aidl 只需定义一次，之后的接口更改，不需要更新 aidl, 可以向下兼容; 

弊端也很明显，失去了 aidl `in， out， inout` 参数跨进程同步修改的能力，object 变成单向的，值传递的，而不是 **引用**。

注意，aidl 的 **引用object** 实质是 binder 实现的实时双向 marshalling, unmashalling, 是两个进程间不同内存地址的对象同步，

跟一般的**Reference引用** 使用同一个进程的同一处内存是不一样的。

举例说，**实现下载及上传文件功能**。按照以往的习惯AIDL接口设计为

```
# 使用 Callback 传输结果，callback 是需要 client 实现的 aidl 接口
oneway api.download(String fileType, String url, String fileName, Callback response);

# 或者 通过aidl 的 in, out 同步参数
# 接口1
api.download(in Request request, out Response response);
# 接口2
api.upload(in Request request, out Response response);
.
.
.
# 接口N
api.xx(x, y, z);
```

现在是，
```
# aidl: remote api 只有一个
oneway api.post(in request, Callback response);

# Request
class Request implement Parcelable {
    int code,
    int apiCode,
    String param
}

# Param json 格式的字符串
param:
{
    fileType: 'string',
    url: 'string',
    fileName: 'string'
}

# java: 每次新加接口，都是 client 加上接口封装 java ApiSdk 
class ApiSdk {
    void download(String fileType, String url, String fileName, Callback response) {
        request.code = xx;
        request.apiCode = ApiCode.DOWNLOAD;
        request.param = {
            fileType,
            url,
            fileName
        };

        api.post(request, response);
    }

    void upload(...) {

    }

    void xx(x, y, z) {

    }
}
```

----


# 适合场景

不需要靠引用传参获取处理结果的情况。

这个设计的api还是多线程的，入口单一却没处理多线程调用，似乎不合理，到底是要多线程还是单线程模型，可以根据具体的业务场景需要在remote 的入口统一处理，比如使用 `MessageQueue` 转成单线程模型。

想到这里才发现，单线程的模型这个就是 `Messenger` 的实现。。。#_#。

我所想到的特性, `Messenger`的设计者都考虑到了。对方真是高手。


----


# 与Messenger的设计对比


## api & Messager aidl: 接口及逻辑层

`Messenger` 用于处理发送请求逻辑，相当于 api.aidl。核心接口及逻辑层。

aidl 部分的实现是在  `Handler` 的内部类 实现了 `IMessenger.aidl` 定义的 `send` 接口

```
private final class MessengerImpl extends IMessenger.Stub {
    public void send(Message msg) {
        msg.sendingUid = Binder.getCallingUid();
        Handler.this.sendMessage(msg);
    }
}

```


对应关系是: `Messenger.send(), IMessenger.send() <--> api.post()`


## Request & Message：数据层


`Message` 作为实体类，存放请求数据，对应 `Request`

**Message**

```
public final class Message implements Parcelable {
    public int what; // 可用于区分接口
    public int arg1; // 简单参数
    public int arg2; 

    /**
     * An arbitrary object to send to the recipient.  When using
     * {@link Messenger} to send the message across processes this can only
     * be non-null if it contains a Parcelable of a framework class (not one
     * implemented by the application).   For other data transfer use
     * {@link #setData}.
     * 
     * <p>Note that Parcelable objects here are not supported prior to
     * the {@link android.os.Build.VERSION_CODES#FROYO} release.
     */
    public Object obj; // 复杂参数

    /**
     * Optional Messenger where replies to this message can be sent.  The
     * semantics of exactly how this is used are up to the sender and
     * receiver.
     */
    public Messenger replyTo; // 相当于 Callback 或 Response 

 
    public int sendingUid = -1;


    // 传输复杂的参数。相当于 Request.params 
     /**
     * Sets a Bundle of arbitrary data values. Use arg1 and arg2 members
     * as a lower cost way to send a few simple integer values, if you can.
     * @see #getData() 
     * @see #peekData()
     */
    public void setData(Bundle data) {
        this.data = data;
    } 
```

Message 还设计了 arg1, arg2 用于只需要传输简单参数的情形。

`Messenger replyTo` 用于 remote 回调结果。client & remote 的来回通讯都是 `Messenger`, 

用同一个机制实现了请求&回调的功能。相比之下，我还特意实现了一个 `Callback.aidl`，有点多余。


我又造了一个不及格的轮子。


# doc

1. [API 常用设计模式：Facade pattern](https://en.wikipedia.org/wiki/Facade_pattern)
