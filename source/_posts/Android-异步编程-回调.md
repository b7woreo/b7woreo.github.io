---
title: Android 异步编程-回调
date: 2019-11-17
typora-root-url: ../../source
tags:
- Android
---

# 为什么需要异步

有始必有因，在说怎么编写异步代码之前我们先要搞清楚为什么需要异步。考虑这样一个用户登陆的业务场景，当用户点击登陆后，首先需要从 SSO 服务器获取 token，然后分别登陆 IM 服务和从业务服务器拉取用户信息，这其中的每一个步骤失败都会导致登陆失败。流程图如下所示：

![](/images/201911171344.png)

根据流程图，设计出如下接口：

``` java
public interface LoginService {

    void login(String username, String password);
}

public interface SsoService {

    Token getToken(String username, String password);
}

public interface ImService {

    void login(Token token);
}

public interface UserInfoService {

    UserInfo getUserInfo(Token token);
}
```

通过组合上述接口实现登陆业务逻辑：

``` java
class LoginServiceImpl implements LoginService {

    private final SsoService ssoService;
    private final ImService imService;
    private final UserInfoService userInfoService;

    LoginServiceImpl(SsoService ssoService, ImService imService, UserInfoService userInfoService) {
        this.ssoService = ssoService;
        this.imService = imService;
        this.userInfoService = userInfoService;
    }

    @Override
    public void login(String username, String password) {
        try {
            Token token = ssoService.getToken(username, password);
            imService.login(token);
            UserInfo userInfo = userInfoService.getUserInfo(token);
            saveUserInfo(token, userInfo);
        } catch (Throwable t) {
            loginExceptionHandler(t);
            throw new BusinessException();
        }
    }

    private void loginExceptionHandler(Throwable t) {
			...
    }

    private void saveUserInfo(Token token, UserInfo userInfo) {
			...
    }
}

```

可以看到，在同步的情况下要描述整个登陆流程并不困难，整个代码非常的简洁，即使是新接触该业务的同学也能通过看看代码很轻松的了解用户登陆经历了哪些步骤。

但是，同步的代码设计有很大的弊端：

1. 不利于充分利用 CPU 资源。
2. 在 Android 平台上，其 UI 框架同其它的平台一样，需要在主线程完成更新 UI、接收用户手势回调等操作。长期占用主线程执行耗时任务会导致应用不能及时响应用户操作而降低用户体验，严重时甚至会出现 ANR。

在接收到用户的点击事件回调后，想要调用上述代码完成登陆操作需要自行完成线程切换的操作，但是该代码最适合在什么类型的线程池中执行只有实现方才知道，无脑的切换线程会导致不必要的开销要不就是导致接口使用与实现耦合。所以，我们需要改造接口，在接口设计时就要以异步的方式来设计，将切换线程这样的操作封装到接口内部，毕竟只有实现方才能分辨逻辑是 **计算密集型** 的还是 **IO 密集型**，以更合理的方式完成执行线程的调度。 

# 基础实现异步的方式-回调

最基础的实现异步的方式就是利用回调，我们首先设计一个通用的接口包含成功和失败的操作：

``` java
public interface Callback<T> {

    void onSuccess(T result);

    void onFailure(Throwable t);
}
```

接着就可以开始将上文中的同步接口全部通过回调的方式改成异步的接口设计：

``` java
public interface LoginService {

    void loginAsync(String username, String password, Callback<Void> callback);
}

public interface SsoService {

    void loginAsync(String username, String password, Callback<Token> callback);
}

public interface ImService {

    void loginAsync(Token token, Callback<Void> callback);
}

public interface UserInfoService {

    void getUserInfoAsync(Token token, Callback<UserInfo> callback);
}
```

同样的我们需要通过组合上述异步接口来实现 LoginService，同时我们还可以做一些小优化：在获得 Token 之后，完全可以同时进行 IM 登陆和用户信息的获取操作以节省登陆耗时。实现代码如下：

``` java
class LoginServiceImpl implements LoginService {

    private final SsoService ssoService;
    private final ImService imService;
    private final UserInfoService userInfoService;

    LoginServiceImpl(SsoService ssoService, ImService imService, UserInfoService userInfoService) {
        this.ssoService = ssoService;
        this.imService = imService;
        this.userInfoService = userInfoService;
    }

    @Override
    public void loginAsync(final String username, String password, final Callback<Void> callback) {
        final Callback<Token> ssoCallback = new Callback<Token>() {
            @Override
            public void onSuccess(final Token token) {
                final Boolean[] status = new Boolean[]{false, false};

                final Callback<Void> imCallback = new Callback<Void>() {
                    @Override
                    public void onSuccess(Void ignore) {
                        synchronized (status) {
                            status[0] = true;
                            if (status[1]) {
                                callback.onSuccess(null);
                            }
                        }
                    }

                    @Override
                    public void onFailure(Throwable t) {
                        handleLoginException(t, callback);
                    }
                };
                imService.loginAsync(token, imCallback);

                final Callback<UserInfo> userInfoCallback = new Callback<UserInfo>() {
                    @Override
                    public void onSuccess(UserInfo userInfo) {
                        saveUserInfo(token, userInfo);

                        synchronized (status) {
                            status[1] = true;
                            if (status[0]) {
                                callback.onSuccess(null);
                            }
                        }
                    }

                    @Override
                    public void onFailure(Throwable t) {
                        handleLoginException(t, callback);
                    }
                };
                userInfoService.getUserInfoAsync(token, userInfoCallback);
            }

            @Override
            public void onFailure(Throwable t) {
                handleLoginException(t, callback);
            }
        };
        ssoService.loginAsync(username, password, ssoCallback);
    }

    private void checkLoginStatus(Boolean[] status, Callback<Void> callback) {
        if (status[0] && status[1]) {
            callback.onSuccess(null);
        }
    }

    private void handleLoginException(Throwable t, Callback<Void> callback) {
			...
    }

    private void saveUserInfo(Token token, UserInfo userInfo) {
			...
    }
}
```

通过回调的方式设计的异步接口要实现于同步代码相同的逻辑，可以看到光中代码行数上就是同步代码的两倍以上，因为这其中不仅仅包含了业务逻辑还包括了异步流程控制逻辑。而且加上层层嵌套的回调，容易形成所谓的地狱回调，导致业务的逻辑被掩埋在其中，对于不熟悉业务的同学想要理清其中逻辑就没有那么简单。

大多数用回调设计的异步接口中并不会对回调接口的执行线程做出一个保证，如果从回调获取数据后，对数据的处理使用必须要在指定的线程中的场景，例如通过回调得到的数据更新 UI，接口的使用方通常都需要自行在回调中完成线程的切换操作，这增加了接口的使用复杂度。

为了让接口便于使用，还需要在接口函数中增加一个 Handler 或 Executor 参数，由接口实现方完成线程切换逻辑。例如在 Android 的 LocationManager 源码中这样：

``` java
public boolean registerGnssStatusCallback (GnssStatus.Callback callback, Handler handler)
```

除了需要考虑回调执行线程的问题，上述代码还有一个缺陷，一旦调用发起了就没有办法中断，也就是没办法对调用的生命周期进行管理。想要解决这个问题也有办法：增加一个 Cancelable 类型作为返回值，外部需要中断时调用 cancel() 方法即可：

```java
interface class Cancelable{
  void cancel();
}
```

可以看到，在不借助任何第三库的情况下是可以通过回调实现异步逻辑的，但是过程十分痛苦，需要考虑过多除业务逻辑外的其它问题。所以，在有现成的专为解决异步问题设计的第三库使用时，不推荐使用这种原始的办法去编写异步代码。