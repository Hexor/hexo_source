---
title: Laravel 使用 JWT 实现 API Auth, 打造用户授权接口
date: 2018-12-13 11:12:26
tags: ['Laravel','Dingo','JWT','API','Auth']
categories: 技术
photos:
- /images/jwt.png
---
Laravel 使用 JWT 实现 API Auth, 打造用户授权接口

<!--more-->
<!-- toc -->


## 前言

本文使用的 Laravel 版本为 5.2


### 本文的目的

1. 介绍基础的服务器用户授权方式 以及 简单的原理概述
2. 介绍基础的 JWT 的使用方式
3. 介绍如何使用 JWT 中间件来进行多个表用户的的权限保护

### Laravel 的授权方式

在编写纯 API 的项目时, 在处理客户端的请求之前, 需要对用户进行认证, 这样就需要解决一个问题, 我们应该如何使用 api 来对用户的身份进行识别?

对于网页来说, laravel 自己有自己的一套工具, 作为初学者我们可以通过

`php artisan make:auth`

来直接生成这一套用户登录和授权的代码. 而对于 纯 api 的用于 app 或者其他网页服务的授权时, 就没有这样一套机制来提供给我们使用了.

但是我们可以提供过使用 JWT 来轻松的实现我们需要的 API 级别的用户认证功能. 在 Laravel 中使用 JWT 我们通常使用 [这个包](https://github.com/tymondesigns/jwt-auth/wiki/Installation)

说到这里, 不妨了解一下 Laravel 中服务器授权方式有哪些.

打开一个 Laravel 项目 ,来到app/Http/Kernel 文件中, 我们可以看到有
```
protected $routeMiddleware = [
    'auth' => \App\Http\Middleware\Authenticate::class,
    'auth.basic' => \Illuminate\Auth\Middleware\AuthenticateWithBasicAuth::class,
    'guest' => \App\Http\Middleware\RedirectIfAuthenticated::class,
];
```

Laravel 已经我们提供了3种授权方式, 并以中间件的方式来提供给我们使用.

* auth

    auth 中间件可以帮我们实现普通的 Web 页面中的用户密码验证, 如果你使用 API 的方式来提供数据接口, 也可以使用 token 的方式来进行验证.

* auth.basic

    也是一种最基础的验证方式, 当你请求一个要求授权的url时, 需要在这个请求的 Header 中加入相关的授权信息, 比如账号和密码, 如果在 web 请求时加上这个中间件, 会在页面中弹出一个输入账号密码的对话框, 输入正确的信息后才能继续访问该页面

* guest

    guest 中间件, 在代码中指向了 `RedirectIfAuthenticated::class` 这个类, 从类名的定义我们就可以看出, 这个中间件的作用就是 如果用户访问了这个中间件下的页面时已经授权成功了, 比如说已经登录过了, 那么就会帮我们跳转到某个另外的 url, 如果没有登录, 就继续当前的请求动作.

上面的这些中间件, 帮我们管理了一部分的用户授权功能, 他们的实现方式大都是通过 Laravel 框架的 Auth 组件(主要是通过 AuthManager 这个类)配合 Guard 类来实现的, 这三种授权中间件的具体实现方式在这里就不做赘述, 之后会写一篇对他们的实现原理的源码解析.

所以按照 Laravel 的思路, 我们的授权功能应该还是用中间件的方式来完成.

### 介绍 JWT
在介绍 JWT 之前, 我们首先介绍一下, 传统的服务器端使用 session 对多用户进行授权的方式. 当然在 session 之前还有 cookie 的方式来保存用户的授权信息在客户机(比如浏览器)上, 不过纯 cookie 的方式过于不安全, 我们就把 cookie 跟 session 一起说.

之所以需要授权机制, 是因为 http 的无状态性(stateless).

也就是说当一个用户发送一次请求, 请求中附带账户名和密码登录成功之后, 如果这个这个用户再次发送一次请求, 服务器是不能知道这个用户是已经登录过的, 这个时候服务器就还需要用户再次提供授权信息, 也就是用户名和密码.

如果客户端能更少的把自己的身份授权信息在网络上传输, 那么客户端就能更大程度上避免自己的身份信息被泄露.

而 session 和 token 都是为了解决此问题出现的.

#### session 原理概述

认证流程

1. 当用户使用用户名和密码登录之后, 服务器就会生成一个 session 文件, session 文件中保存着对这个用户的授权信息, 这个文件可以储存在硬盘/内存/数据库中.
2. 同时还要生成一个对应这个 session 文件的 sessionid, 通过 sessionid 就能够找到这个 session 文件.
3. 然后将 sessionid 发送给客户端, 客户端就将 sessionid 保存起来, 保存的方式有很多种, 目前大多情况是通过 cookie 来保存 sessionid.
4. 保存之后, 当客户机以后再向服务器发送请求的时候, 请求携带上 sessionid, 这样服务器收到 sessionid 之后, 自己就会在服务区上查找对应的 session 文件, 如果查找成功, 就会得到该用户的授权信息, 从而完成一次授权.

session 的出现解决了一部分的问题, 但随着时间的推移和互联网的发展, 一些缺陷也随之暴露出来, 比如但不仅限于以下几点

* 随着用户量的增加, 每个用户都需要在服务器上创建一个 session 文件, 这对服务器造成了压力
* 对于服务器压力的分流问题, 如果一个用户的 session 被存储在某台服务器上, 那么当这个用户访问服务器时, 用户就只能在这台服务器上完成授权, 其他的分流服务器无法进行对这种请求进行分流
* 同样也是 session 存储的问题, 当我们在一台服务器上成功登录, 如果我们想要另外的一台别的域名的服务器也能让用户不登录就能完成授权, 这个时候就会有很多麻烦
* CSRF 攻击的防范, 这点目前还没太懂, 不过可以先不做了解

为了解决此类问题, token 应运而生了.

#### Token 原理概述

认证流程

1. 客户端发送认证信息(一般就是用户名/密码), 向服务器发送请求
2. 服务器验证客户端的认证信息, 验证成功之后, 服务器向客户端返回一个 加密的token(一般情况下就是一个字符串)
3. 客户端存储(cookie, session, app 中都可以存储)这个 token, 在之后每次向服务器发送请求时, 都携带上这个 token
4. 服务器验证这个 token 的合法性, 只要验证通过, 服务器就认为该请求是一个合法的请求


#### JWT 概述

token 只是一种思路, 一种解决用户授权问题的思考方式, 基于这种思路, 针对不同的场景可以有很多种的实现. 而在众多的实现中, JWT(JSON Web Token) 的实现最为流行.

JWT 这个标准提供了一系列如何创建具体 token 的方法, 这些缘故方法和规范可以让我们创建 token 的过程变得更加合理和效率.

比如, 传统的做法中, 服务器会保存生成的token, 当客户端发送来token时, 与服务器的进行比对, 但是 jwt 的不需要在服务器保存任何 token, 而是使用一套加密/解密算法 和 一个密钥 来对用户发来的token进行解密, 解密成功后就可以得到这个用户的信息.

这样的做法同时也增加了多服务器时的扩展性, 在传统的 token 验证中, 一旦用户发来 token, 那么必须要先找到存储这个 token 的服务器是哪台服务器, 然后由那一台服务器进行验证用户身份. 而 jwt 的存在, 只要每一台服务器都知道解密密钥, 那么每一台服务器都可以拥有验证用户身份的能力.

这样一来, 服务器就不再保存任何用户授权的信息了, 也就解决了 session 曾出现的问题.


<img src="http://wx1.sinaimg.cn/large/9bfd7269gy1fjwwftyvv7j21120ksgnk.jpg"  align=center/>


### Laravel 推出了自己的用户认证模块 "Passport", 应该使用吗?

在这篇文章完成之前, Laravel 又推出了一个用户认证的高级实现工具, 也就是 Laravel Passport, 毕竟是 Laravel 家族的产品, 所以刚发布的时候我也想要去使用他, 研究一番之后才发现, Passport 并不是适合所有场景的, 下面我们就来对比一下.

这两者都是用来做用户认证的. 但是有几点比较重要的不同点.

相比于 jwt, passport 是一个大的多的抽象层, 他被设计成一个完全成熟的(同时也要容易配置和使用) Oauth2 服务来使用. 因为在设计之初, passport 的目标就被设计的非常专一, 所以在别的使用场景下, passport 的功能也很难被复用. 而且, passport 内部有一部分使用了 jwt 来实现认证.

而 jwt 的设计目的更倾向于, 是一个简单的用户认证方式. 当然jwt能实现的功能是十分强大的, 但他跟 Oauth 并没有什么关系.

所以一般来说, 你能用jwt完成的需求, 用 passport 一定也能完成. 但是并不推荐使用passport, 除非你有 Oauth相关的需求. 如果你真的需要一台支持 Oauth 的服务器, 那么我完全推荐你使用 Passport, 因为他非常容易配置.

还有一点需要注意, 服务器允许其他 Oauth 用户登录, 跟服务器自身提供Oauth 服务是2个概念. 如果你想要的只是类似于 "使用 Google 帐号登录" 这种功能, 那么你应该使用 Laravel/socialite. 服务器自身提供 Oauth 服务意味着, 别的网站的开发者可以使用你提供的用户认证信息来登录到他们的服务器上.

### 如果不想使用 JWT 和 Passport, Laravel 能快速实现使用 token 的用户认证机制吗?

---- 提纲 ( 下面应该做什么 )
把一个新的项目改成能使用 token 登录, 注册, 应该做什么, 进行一步步的讲解.



答案是可以的. Larave 自身的代码已经实现了大部分功能, 所以我们只要稍作修改, 就能实现基本的 token 认证了.

步骤如下:

1. 新建一个 Laravel 5.2.* 的项目
2. 打开 `database/migrations/2014_10_12_000000_create_users_table.php` 这个 migration 文件, 我们需要更改 user 表的结构
3. 我们需要为 user 表添加 api_token 字段, 也就是说我们的 token 是保存在数据库中的, 在合适的位置, 添加一行

  ```
  $table->string('api_token', 60)->unique();
  ```

4. 配置好数据库, 通过 `php artisan migrate` 命令生成 user 表
5. 在user表中, 随便添加一条记录, 只要保证 api_token 这个字段设置为 123456 即可. 这样我们就生成了一个用户, 等下就可以使用 123456 这个token 值来登录了.
6. 返回到 路由文件 `routes.php`, 在里面添加一条测试路由, 并将其 用 laravel 的中间件保护起来

  ```
  Route::group(['middleware' => 'auth:api'], function () {
      Route::get('/t', function () {
          return 'ok';
      });
  });
  ```

  在此处, 使用的是 `auth` 中间件, `:api` 的意思是说, 我们指定了 auth 中间件的 driver 为 api, 而如果不指定的话, driver 默认使用的是 web. 关于认证的配置文件 `config/auth.php` 中各项配置的介绍, 再次就不做赘述, 可以参考这个 [视频](https://www.youtube.com/watch?v=iKRLrJXNN4M复杂) 中的讲解.

7. 修改 `app/Http/Middleware/Authenticate.php` 文件, 精简一下 `handle` 这个方法, 精简之后成为下面的样子

  ```
  public function handle($request, Closure $next, $guard = null)
  {
      if (Auth::guard($guard)->guest()) {
          return response('没有设置token.', 401);
      }

      return $next($request);
  }
  ```
8. 做了以上修改之后, 当我们以 `/t` 这个 url 路径向服务器直接发起请求时, 服务器就会返回一个 401 错误, 并且会返回一条 `'没有设置token.'` 这样的消息, 这也是我们之前在 `handle()` 方法中设置的. 也就是说 `/t` 已经被我们的 `auth` 中间件保护起来了. 如果想要我们的请求能够正常通过这个中间件, 就要提供 token.
9. 由于我们之前在 user 表中添加了一条 api_token 为 123456 的数据, 所以现在我们再次向服务器请求 `/t`, 但是这次我们加入 api_token, 也就是

    `.../t?api_token=123456`

    正常情况下, 服务器就会返回 'ok' 了, 这也就是说明, auth 中间件允许这个请求通过. 而当我们把 123456 修改为其他值时, 这个请求也是无法通过 auth 中间件的.


当然, 上面只是最简单的实现方式, 在实际的应用中会比这个更加复杂.


## 使用 JWT / Why JWT
简单介绍完了 JWT 之后, 接下来我们就简单看一下在实际场景中 JWT 的应用.

安装和配置 JWT https://github.com/tymondesigns/jwt-auth/wiki/Installation


### 创建用户

Controller 中的方法
    ```
    public function register(Request $request)
        {
            $newUser = [
                'email' => $request->input('email'),
                'name' => $request->input('name'),
                'password' => bcrypt($request->input('password'))
            ];

            $user = User::create($newUser);
            $token = JWTAuth::fromUser($user);

            return responseSuccess($token);
        }
    ```

### 用户登录

Controller 中的方法
```
public function authenticate(UserLoginRequest $request)
{
    $credentials = $request->only('email', 'password');

    try {
        if (! $token = JWTAuth::attempt($credentials)) {
            return responseError(-1, "邮箱或者密码错误");
        }
    } catch (JWTException $e) {
        return responseError(-3, "token 无法生成");
    }

    return responseSuccess(compact('token'));
}
```
## #获取用户

```
$user = JWTAuth::parseToken()->authenticate();
```

### 用户登出

Controller 中的方法

```
public function logout()
    {
        JWTAuth::setToken(JWTAuth::getToken())->invalidate();

        return responseSuccess();
    }
```

### 使用 JWT Auth 中间件

按照官方文档说明在 `app/Http/Kernel.php` 中的合适位置添加

```
'jwt.auth' => 'Tymon\JWTAuth\Middleware\GetUserFromToken',
'jwt.refresh' => 'Tymon\JWTAuth\Middleware\RefreshToken',
```

之后便能够在路由中使用 `jwt.auth` 中间件来保护指定的url了.

对于被 `jwt.refresh` 保护的 url 来说, 用户一旦访问这个 url, 那么此次的访问是能够通过的, 但是同时用户当前的 token 就会失效


比如, 现在如果想要正常访问 `/t` 这个 url, 就必须在请求中附加 `token` 值才可以, 而 `token` 值是在上面的用户登录方法 `authenticate` 中获取到的

```
Route::group(['middleware' => 'jwt.auth'], function () {
    Route::get('/t', function () {
        return 'ok';
    });
});
```

假设我们登录成功得到的 token 为 `q1w2e3`, 那么就可以是通过下面这个 `url` 来正常访问

`.../t?token=q1w2e3`

除了在 url 中附带 token 之外, 你还可以在请求的 header 中附带 token 值来完成 jwt.auth 的验证. 具体实现方法可以在 JWT 组件中的 `JWTAuth.php` 文件中的 `parseToken()` 方法中找到.

比如, 我们可以设定这样的一个 header:

key 为 "authorization"
value 为 "bearer q1w2e3"

这样就不需要在 url 中附带 token 值了, 也能通过中间件的验证.



### 异常消息处理

当遇到异常情况时, 比如用户账户密码错误, 或者 token 过期时, JWT 默认会返回基础的错误代码和错误消息, 如果需要修改这些设定, 也是十分方便的.

在上面, 我们已经添加过 `'jwt.auth' => 'Tymon\JWTAuth\Middleware\GetUserFromToken',` 这个中间件, 而这个中间件也会负责应对异常和错误的处理, 所以来直接看一下 `GetUserFromToken` 的源代码,

```
public function handle($request, \Closure $next)
{
    if (! $token = $this->auth->setRequest($request)->getToken()) {
        return $this->respond('tymon.jwt.absent', 'token_not_provided', 400);
    }

    try {
        $user = $this->auth->authenticate($token);
    } catch (TokenExpiredException $e) {
        return $this->respond('tymon.jwt.expired', 'token_expired', $e->getStatusCode(), [$e]);
    } catch (JWTException $e) {
        return $this->respond('tymon.jwt.invalid', 'token_invalid', $e->getStatusCode(), [$e]);
    }

    if (! $user) {
        return $this->respond('tymon.jwt.user_not_found', 'user_not_found', 404);
    }

    $this->events->fire('tymon.jwt.valid', $user);

    return $next($request);
}
```

代码中的 `$this->respond` 实现如下

```
protected function respond($event, $error, $status, $payload = [])
{
    $response = $this->events->fire($event, $payload, true);

    return $response ?: $this->apiResponse($status, $error);
}
```
可以看到, 在有异常情况时, JWT 会产生一个 Event , 之后再返回错误消息和错误码返回给请求调用者.
所以可以通过捕捉这个 event, 来直接按我们想要的方式进行错误处理.

所以我们就可以在 `app/Providers/EventServiceProvider.php` 的 `boot()` 方法中加入对 JWT 事件的监听, 并且直接返回数据

```
public function boot()
    {
        parent::boot();

        Event::listen('tymon.jwt.absent', function () {
            return response()->json([
                'code' => -100,
                'msg' => '未提供 Token, 请重新登录',
                'data' => '',
            ]);
        });

        Event::listen('tymon.jwt.invalid', function () {
            return response()->json([
                'code' => -100,
                'msg' => '无效的 Token, 请重新登录',
                'data' => '',
            ]);
        });



        Event::listen('tymon.jwt.expired', function () {
            return response()->json([
                'code' => -100,
                'msg' => 'Token 已经过期, 请重新登录',
                'data' => '',
            ]);
        });



        Event::listen('tymon.jwt.user_not_found', function () {
            return response()->json([
                'code' => -101,
                'msg' => '没有找到该用户',
                'data' => '',
            ]);
        });
    }
```

### 定制 JWT 中间件

在实际应用场景中, 有时 `'Tymon\JWTAuth\Middleware\GetUserFromToken'` 并不能完全满足我们的需求, 此时就可以仿照 JWT 创建自己的中间件了.

比如, 我们可以创建一个新的中间件 `app/Http/Middleware/CustomJWTAuth.php`, 里面的内容直接从 JWT 的 `GetUserFromToken` 复制即可, 复制完了记得修改 namespace 为

`namespace App\Http\Middleware;`

之后, 在 Kernel 中的 $routeMiddleware 中添加我们刚刚创建好的中间件

`'custom.jwt.auth' => \App\Http\Middleware\CustomJWTAuth::class,`

最后, 在路由中就能正常使用我们自己创建的中间件了.

```
Route::group(['middleware' => 'custom.jwt.auth'], function () {
    Route::get('/t', function () {
        return 'ok';
    });即可
});
```

此时, 再去 `CustomJWTAuth.php` 完成你想要的定制即可.

### 动态切换 JWT 指定的用户模型

在每次 Laravel 服务器接收到请求之后, 都会开启一次生命周期. 在生命周期的初始几个阶段, 程序会读取 config 文件夹下的配置文件, 并且使用这些配置文件, 这其中就包括 JWT 的一些设置. 利用这个特性, 以及 JWT 的 token 不在服务器保存的特点, 我们就能实现这一功能.

以这样一个场景举例, 服务器上有 admin 和 user 2个table, 那么如果我们想要用 JWT 中间件分别保护两种不同类型的用户的访问权限, 就可以很轻松的实现.

1. 在 `config/jwt.php` 和 `config/auth.php` 中, 用户认证的默认模型都是 `user`, 所以我们只要能动态修改 user 为 admin, 就能让 JWT 同一时间保护不同的访问权限.

2. 此时, 我们需要创建2个中间件,

  `'custom_user.jwt.auth' => \App\Http\Middleware\CustomUserJWTAuth::class,`

  `'custom_admin.jwt.auth' => \App\Http\Middleware\CustomAdminJWTAuth::class,`

  CustomUserJWTAuth 与 CustomAdminJWTAuth 都先复制 JWT 的代码即可.

3. 之后, 在 `CustomAdminJWTAuth.php` 中作出以下调整, 在 `handle()` 的开始的部分加入下面内容

  ```
  public function handle($request, \Closure $next)
  {
      Config::set('jwt.user', 'App\Models\Admin');
      Config::set('auth.providers.users.model', \App\Models\Admin::class);
  ```

  在请求通过我们的 Admin 用户认证中间件 `custom_admin.jwt.auth`, 上面这的两行就可以把 JWT 的用户模型从 user 切换到 admin.

4. 上面只是说了在中间件中的设置. 当在用户 注册/登录/登出 等场景时, 同样需要临时的修改JWT的默认用户模型, 比如在用户登录(获取token)时, 我们就需要重新在 Controller 中写一个新的登录方法, 供 Admin 的角色进行登录, 我们可以直接复制上面贴出来的代码, 再进行修改

    Controller 中的方法

    ```
    public function authenticateForAdmin(AdminLoginRequest $request)
    {

        // 切换 JWT 用户模型
        Config::set('jwt.user', 'App\Models\Admin');
        Config::set('auth.providers.users.model', \App\Models\Admin::class);

        $credentials = $request->only('email', 'password');

        try {
            if (! $token = JWTAuth::attempt($credentials)) {
                return responseError(-1, "邮箱或者密码错误");
            }
        } catch (JWTException $e) {
            return responseError(-3, "token 无法生成");
        }

        return responseSuccess(compact('token'));
    }
    ```

5. 示例路由
  ```
  Route::group(['middleware' => 'custom_user.jwt.auth'], function () {
      Route::get('/user_test', function () {
          return 'user_test ok';
      });
  });
  ```

  ```
  Route::group(['middleware' => 'custom_admin.jwt.auth'], function () {
      Route::get('/admin_test', function () {
          return 'admin_test ok';
      });
  });
  ```

  如果一切设置的正确, 当我们拿着两次不同登录(一次通过 user 模型登录, 一次通过 admin 模型登录)获取到的 token 分别访问上面的两个路由, 会发现每个 token 只能正常访问上面其中一个路由, 访问另外一个路由则会提示 token 无效的错误, 这时由于每个 token 都是针对其特定的模型生成的缘故.


## 最后

关于 JWT 的设置项还有很多, 比如 token 过期时间等, 这些都可以在 `config/jwt.php` 中进行配置. 具体配置方法请参照官方文档 https://github.com/tymondesigns/jwt-auth/wiki/Configuration

本文没有介绍关于 Laravel 中 JWT 的基础安装和使用方法, 这些都可以在 https://github.com/tymondesigns/jwt-auth/wiki 中找到.

如有疑问或纰漏, 欢迎留言.