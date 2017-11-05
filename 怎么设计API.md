# 单个API

评定的标准
1. *简洁易用*： 参数不超过4个，不需要注释就知道用途及使用方式
2. *易扩展*：更改逻辑不需要动原先的代码，只需要注入新的实现。  
比如注入新的接口实现。一般的做法是，预留一些可重载的接口，或提供注入的接口。  
例如OkHttp 提供了 addIntercetor 更改 Request, Response 流程的逻辑; Gson 的 addTypeAdapter 自定义类型转换。

如何检验？记住 ‘rule of three’。思考至少3个调用场景，3个扩展的情况。参考 Qt API 设计
+ [Qt: API Design Principles](https://wiki.qt.io/API_Design_Principles)
+ [The Little Manual of API Design](http://www4.in.tum.de/~blanchet/api-design.pdf)  

例如登录接口，用户输入用户名，密码登录，对应的后台接口需要 用户名，加密sign，时间戳等。  

## 标准: 简洁易用  
前端的API有以下几种写法：

### 1. 大对象传输参数型  
```java
public class LoginParam {
    String password;
    String userName;
    String sign;
}
public void login(LoginParam param, Callback callback);
```

如果参数少于5个，这种方式更像是过度封装了，多一个不必要的**参数对象**。如果参数过多，使用 *Builder Pattern* 更合适。  
弊端是，阅读&使用不方便，需要配合文档，否则使用者容易漏掉某些必选参数。  
这种方式在某些动态语言中运用的较多，因为书写及其方便优雅，在PHP，JavaScript 中尤其常见. 例如  
```javascript
// JavaScript
login({
    'user': 'xechoz'
    'password'： 'hello',
    'from': 'mainPage',
    'autoLogin': true
}, result => {
    
});
```
参见[Node.js Api: http.request(options[, callback])](https://nodejs.org/api/http.html#http_http_request_options_callback), 其options包含了 10+ 个参数。

### 2. 罗列参数型

```java

// 情形1：把计算参数的逻辑丢给调用者，如 sign = makeSign(...)
public void login(String userName, String sign, Callback callback)；
public static String makeSign(String userName, String passoword, long timeStemp, ...);

// 情形2：用最少的参数，不暴露底层接口细节
public void login(String userName, String password, Callback callback);
```
简洁，易用。但是实际的开发中，只有一个逻辑的接口是比较少见的。更多的情况是，还有一个前置或后置的依赖流程接口，  
这时login的接口很可能就必须带上相应的参数了，虽然这些不是**登录**所必须的。

### 3. 长龙参数型  
```java
// 情形1：长参数列表
/**
* 登录。from, isForceLogin, isAd, adId 用于统计用户来源
* ...
* @param isFromAd 是否为广告页面带来的登录
* @param adId 广告id。只有 isFromAd = true 时才需要。默认0
**/
public void login(String userName, String password, Callback callback, Activity from, boolean isForceLogin, boolean isAd, int adId)

// 情形2： 分离接口
// 还是分为两个接口？让调用者保证调用login 之前调用 postLoginEvent?
public void login(String userName, String password, Callback callback);
public void postLoginEvent(String userName,  Activity from, boolean isForceLogin, boolean isAd, int adId);
```
实际的开发场景中，**长龙参数型**并不少见，因为不断添加需求，不同的业务逻辑耦合在同一个接口里，  
如登录的时候需要统计用户来源，到底是写在同一个接口里，还是另外写一个接口，让使用者调用login之前，先调用统计接口？

最后，似乎没有一种实现方式符合*简洁易用*的评定标准。只能根据具体的参数依赖，数量等选择当前最合适的API实现方式，  
不久之后，随着业务的更改，这种实现方式就会显得臃肿，不合适，需要一次重构。这就是理论与现实的距离。

## 标准: 扩展性
易扩展的基础是，基于接口编程。依赖接口，并提供一个默认的接口实现，后期的更改&扩展可以注入新的接口实现。  

```java
public interface ILogin {
    void login(String userName, String password);
}

class DefaultLogin implements ILogin {
    ...
}

class ApiLogin implements ILogin {
    private ILogin loginImpl = new DefaultLogin();

    public void setLoginImpl(ILogin loginImpl) {
        this.loginImpl = loginImpl;
    }

    @Override
    public void login(String userName, String password) {
        loginImpl.login(userName, password);
    }
}

// .. inject you implement
apiLogin.setLoginImpl(new MyLoginImpl());
```

又或者使用 `template/hook` 模式。这种适合在主流程不改动的情况下，扩展某一个步骤的实现。如登录完成执行的操作。  
如Android View 的 draw -> onDraw, layout -> onLayout，使用者通过 hook 扩展自定义的行为。
```java
@Override
protected void onPreLogin() {
    // 保存登录来源信息
}

@Overide
protected void onLogin(UserInfo info) {
    super.onLogin(info);

    if (info != null && info.isLogin()) {
        // 保存统计信息
    } else {
        // 发送登录失败
    }
}

@Override
protected void onDraw(Canvas canvas) {
    // 自定义的绘制内容
}
```

# API 集
我把 API 设计风格分为两种： 分布式，集中式  
假设现在有个需求，账户模块。从业务逻辑上可以分为，登录注册逻辑，账户信息，数据存储 等子模块。

## 分布式
顾名思义，分布式的api是各自独立的，尽量减少不必要中间依赖模块。比如，登录模块，注册模块 可以各自独立，不必有一个中间模块把两者强行耦合在一块。

```java
class ApiLogin {
    public void login() {...}
}

class ApiData {
    public void save(data) {...}
}

class ApiSession {
    public void setSession(String token, int timeOut) {...}
}
```

## 集中式
可以理解为把分布式的接口集中到一块的方式。这是最常见的设计方式，也是最容易被滥用的方式。用得好，接口可以很易用，因为入口唯一；用得烂，接口容易变得臃肿。 

集中式的设计分为:
+ *一层设计：core module 暴露所有api*   
core module 实现所有的 api 或者组合 子module，具体api由 子module 实现。  
适合api少量的情况，如果api过多， core 模块容易臃肿，越到后期维护学习成本越大。一般我设计的API类，API 数量不超过10个, 以5个为标准；  
另外，这种方式，每次添加接口都要改动 core api, 不相干的api也因为这个 core 类而存在被改动的风险。比如 core 包含了 登录，支付,   
现在需要添加一个数据保存，而 登录支付 跟 数据保存 这些业务逻辑或api本应各自独立的。
+ *二层设计：core api 暴露 sub module + 各个 sub module 暴露&实现各自的api*  
如果某个 sub module 包含的api 很多，可能也会采用分层的方式包含自己的子api层。递归地包含更多的层级。  
core api 暴露这几个子模块，具体的api由子模块暴露。

### 代码示例
```java
// 一层设计：
class CoreApi {
    public void login() {
        // login implement here
        // or  
        // apiLogin.login();
    }

    public void setUserInfo(UserInfo info) {...}
}

// 二层
class CoreApi {
    ApiLogin getApiLogin() {
        return apiLogin;
    }

    ApiUser getApiUser() {
        return apiUser;
    }
}

class ApiLogin {
    public void login() {
        // login implements
    }
}

class ApiUser {
    public void setUserInfo(UserInfo info) {}
    //...
}
```

# 注意事项

1. observer pattern 的 API 注意内存泄露。考虑 WeakRefrence 等。最常见的如注册某个事件监听。
2. Singleton & static 谨慎使用。释放不及时，或者 static 对象过多，都是OOM的罪魁祸首。
