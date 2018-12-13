---
title: 从 1 行代码开始，带你系统性的理解   Laravel  Service Container 的核心概念
date: 2016-12-04 23:11:29
<!-- top: 2 -->
tags: ['Laravel']
categories: 技术
photos: 
- /images/Laravel-core-concepts.png
---
帮助大家 ( 特别是新手 )，从1行代码开始，系统性的理解 Laravel IOC 系统的架构

<!--more-->
<!-- toc -->

## 概览

### 本文的目的
帮助大家 ( 特别是新手 )，从1行代码开始，系统性的理解   Laravel   IOC 系统的架构，从而了解 

* Service Container
* Service Providers
* Facades
* Contracts 

上述4点相关的核心概念。

之所以说是系统性，是因为我不打算像官方文档一样，把上面这4点分成4个部分来说，因为实在是太难懂了。

### 阅读之前
在阅读本文之前，你**必须**了解

* 什么是 IOC
* 如何构造一个IOC
* IOC 为系统的架构提供了哪些好处

关于 IOC 的问题你可以从[这篇文章](https://laravel-china.org/topics/789)中得到很好的答案。

当然，你最好也能从[这里](https://hexor.github.io/2016/11/30/Laravel-%E8%AF%B7%E6%B1%82%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F/)了解一下 Laravel 请求的生命周期。

### 阅读之后
希望大家能通过阅读本篇文章对上述4个核心概念有更好的理解，至少。。别人问你你能答出来这4点都是干什么的

</br>
</br>
## 正文


</br>
### 一个基础版的 IOC Container
也许你以前就知道什么是 IOC， 也许你是从上面那篇文章才开始对 IOC 有懵懂的认识。 

现在我们都从同一个起点出发. 就是再来重新再来写一个基础版的 IOC Container.

为什么要写这个基础版的 IOC Container 出来呢， 因为 Laravel   框架的 IOC Container 原理上跟上面的这个基础版是一模一样的, 包括方法的名字, 参数的传入方式等.

而且在下文，当我使用 `bind` 和 `make` 2个词的时候，我指的就是 Container 中的这2个方法。

```php 
class Container
{
  protected $binds;

  protected $instances;

  public function bind($abstract, $concrete)
  {
    //Todo: 向 container 添加一种对象的的生产方式

    //$abstract: 第一个参数 $abstract, 一般为一个字符串(有时候也会是一个接口), 当你需要 make 这个类的对象的时候, 传入这个字符串(或者接口), 这样make 就知道制造什么样的对象了
    //$concrete: 第二个参数 $concrete, 一般为一个 Closure 或者 一个单例对象, 用于说明制造这个对象的方式

    if ($concrete instanceof Closure) {
      $this->binds[$abstract] = $concrete;
    } else {
      $this->instances[$abstract] = $concrete;
    }
  }

  public function make($abstract, $parameters = [])
  {
    //Todo: 生产一种对象

    //$abstract: 在bind方法中已经介绍过
    //$parameters: 生产这种对象所需要的参数

    if (isset($this->instances[$abstract])) {
      return $this->instances[$abstract];
    }

    array_unshift($parameters, $this);

    return call_user_func_array($this->binds[$abstract], $parameters);
  }
}
```


### 为什么理解 IOC Container 对于理解 Laravel 架构是如此的重要？
因为在 Laravel 中，你所能使用到的 Laravel 的特性和功能几乎全部是由 IOC Container 实现的。

比如

```php
Cache::get('key'); 
Route::get('/', 'HomeController@index');
```
Cache 和 Route 都是通过把他们各自的实现类 `bind` 到某个  Laravel 的 Container 后，那个 Container 所 `make` 出的一个实例。

也许你现在有些疑问：到底是在哪进行 `bind` 操作的，又是 `bind` 到哪一个 Container 了，这个 Container 又是在什么地方 `make` 了他们？ 

这些问题现在你都不需要知道，看到后面你会有答案。现在你只需要知道

**IOC Container 组成了  Laravel 的架构，是  Laravel  的核心机制。**

在 Laravel 中，他们把这个叫做  Laravel  的 Service Container

Service Container 是今天的第一个角色，主角。

</br>


### 该如何使用 Laravel 的 IOC Container（ Service Container ）？

如果想要使用 Laravel 的 IOC Container，也就是说想要用 IOC 的机制去 `make` 某种对象, 那么你就必须先 `bind` 这个对象的类到   Laravel    的 IOC Container 中, 才能把这种对象 `make` 出来。

至此，可以请出今天第二个角色了，Service Provider

为啥 Service Provider 突然蹦出来了呢，因为在  Laravel  中，我们大体可以上有2种方式去使用 IOC Container：

1. 通过 Service Provider 来使用IOC Container
2. 不通过 Service Provider 直接使用 IOC Container

大多数情况下，我们使用第一种方式。为什么呢，我们先从第二种开始说起。

#### 如何不通过 Service Provider 直接使用 IOC Container？

Laravel 有一个核心类，叫做 Application，这个继承了 Container，所以很显然，这个类是一个 IOC Container

```php
// 该类的命名空间
namespace Illuminate\Foundation; 

// 该类的声明
class Application extends Container implements ApplicationContract, HttpKernelInterface
```

在  Laravel 程序启动的时, 在 bootstrap/app.php  里面实例化了这个类，并把这个类的实例交给了 $app 。如果你了解  [Laravel  请求的生命周期](https://hexor.github.io/2016/11/30/Laravel-%E8%AF%B7%E6%B1%82%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F/)，那你就会对这里的程序流程更清楚一些，不过不了解也没有大碍。

```php
// 在 bootstrap/app.php 中实例化了该类
$app = new Illuminate\Foundation\Application(
    realpath(__DIR__.'/../')
);

```
比如我们现在需要不通过 Service Provider 直接使用 IOC Container ，就是要使用上面的这个 IOC Container， 也就是 $app 。

很简单， 我们要做的第一件事就是 `bind`， 第二件事就是 `make`

新建一个 Post 类

```php
<?php

namespace App;

use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
  public $d = "123";
}

```

然后 `bind`

```php
App::bind('post', function ($app) {
  return App::make('App\Post');
});
```
最后 `make`

```php
$post = App::make('post');
return $post->d; //将会返回 "123"
```
就像这样，完全不使用 service provider，我们也完成了对  Laravel  IOC Container 的使用。

也许你会问，为什么我们直接用了 App 而不是 $app，这是因为 Larave 使用了 Facades 的特性，来让你在程序的各处都能方便的得到 $app，或者说 Application 类的这个实例。

也许你又会问，Facades 又是个啥？回答你，现在先别管，等下就知道。不过提前告诉你，Facades 也是今天的角色之一。

#### 为何大多数情况都通过 Service Provider 来使用 IOC Container？
我们知道，有时候我们的类、模块会有需要其他类和组件的情况，为了保证初始化阶段不会出现所需要的模块和组件没有注册的情况，laravel 将注册和初始化行为进行拆分，注册的时候就只能注册，初始化的时候就是初始化。拆分后的产物就是现在的 服务提供者。

可以想象这样一个场景, 你要绑定3个类 A B C 到 IOC Container 中。 A，B，C 都是非常复杂的类。在 `bind` A 时, 引用了一个类 B 的实例, 那么想要获得类 B 的实例，就需要 B 已经被 `bind`，只有这样，我们的 IOC Container 才有能力 `make` 出一个 B 的实例。 而在 `bind` B 时, 恰好又需要 C 的实例. 

如果是这样的逻辑, 那么在 `bind` A B C时, 就必须手动的严格安排 `bind` 的次序, 而且这只是3个类的情况, 如果有几十个类的话, 人工已经无法完成了.

而这时就需要 Service Provider 的作用了。

引用一段别人的话
>我们知道，有时候我们的类、模块会有需要其他类和组件的情况，为了保证初始化阶段不会出现所需要的模块和组件没有注册的情况，laravel 将注册和初始化行为进行拆
分，注册的时候就只能注册，初始化的时候就是初始化。拆分后的产物就是现在的 Service Provider。

#### 如何通过 Service Provider 来使用 IOC Container？

关于如何通过 Service Provider 来使用 IOC Container，我通过下面的例子来说明

##### Laravel 的发明者是如何通过 Service Provider 来使用 IOC Container 为 Laravel 框架添加特性和功能的？

我们从这行代码说起，这里终于是标题中提到的那 **1** 行代码啦。

```php
Route::get('/', 'HomeController@index');
```

你是无法找到对 Route 类的声明的，为啥呢，因为使用了别名。别名是  PHP  的一个特性（ class_alias 方法 ）。

Route 是如何配置成为别名的呢， 在 `app/config/app.php` 中， 我们可以看到  Laravel  把所有的别名配置都放在了这个数组中

```php
'aliases' => [
    ...
    'Route' => Illuminate\Support\Facades\Route::class,
    ...
```

更细节的，关于 Laravel  是如何把这个数组里的别名都配置好的，本文就不再提及，在源代码中搜索 class_alias 就可以看到相关内容。

继续往下说，我们看到 Route 实际上是代表了 `Illuminate\Support\Facades\Route::class` 这个类， 我们找到这个类

```php
namespace Illuminate\Support\Facades;

/**
 * @see \Illuminate\Routing\Router
 */
class Route extends Facade
{

    protected static function getFacadeAccessor()
    {
        return 'router';
    }
}
```

看到这个类之后，并没有找到之前 Route 调用的 `get` 方法，此时我们再看里面的这行注释，`@see \Illuminate\Routing\Router`，他提示我们去找这个位置，那我们就去找一下，我们又发现了一个 Router 类，而这个 Router 类中，是有 `get` 方法的，看起来这里似乎就是 Route 的真实身份了。

```php
namespace Illuminate\Routing;
class Router implements RegistrarContract {
    ...
    
    /**
     * Register a new GET route with the router.
     *
     * @param  string  $uri
     * @param  \Closure|array|string|null  $action
     * @return \Illuminate\Routing\Route
     */
    public function get($uri, $action = null)
    {
        return $this->addRoute(['GET', 'HEAD'], $uri, $action);
    }
    
    ...
}
```

那  Laravel  是如何为 `Illuminate\Support\Facades\Route::class` 这个类找到他的真实身份的呢？

此时，正式请出我们今天的三号角色，Facades。


*先纠正一下大家的读音，有的人可能会把这个词读做 fei kei de，其实这个词读作 [fə'sɑd]，拼音差不多是 fo sa de*
*词典中，facade 是一个名词。翻译为: 正面；表面；外观*
*读音:   英  [fə'sɑːd]   ；美  [fə'sɑd]*


**Facade 的作用是用一个简单易记的语法，让你从  Laravel  的 IOC Container 中方便的 `make` 出你想要的类的对象。**

之前我们看到 `class Route extends Facade`，说明 Route 也是一个 Facade，那这个 Route 的作用就是：

让我们通过这种简单的语法 Route::get(...)，去 Laravel  的 IOC Container 中方便的 `make` 出上面的 Route 的真实身份 `Router`。

##### Facade 是如何使用的？

Facade 是如何做到上面所描述的事情的呢？ 下面进行讲解。

首先， `class Route extends Facade`， Route 继承自 Facade 类，Route 类又调用了静态的 `get` 方法，我们在 Route 类，或者是他的父类 Facade 中都是无法找到这个 `get` 方法的。 但是在 Facade 类中，我们可以发现有一个 `__callStatic()` [魔术方法](http://blog.csdn.net/liutingxu1/article/details/17655049)，这个方法的作用就是：如果你想要调用的静态方法在类的定义中并没有声明，那么就会执行 `__callStatic()`。在我们当前的情景中，静态方法 `get` 并没有被声明，那么当然，我们的类就会转而调用 `__callStatic()` 。

```php
public static function __callStatic($method, $args)
{
    $instance = static::getFacadeRoot();

    if (! $instance) {
        throw new RuntimeException('A facade root has not been set.');
    }

    switch (count($args)) {
        case 0:
            return $instance->$method();
        case 1:
            return $instance->$method($args[0]);
        case 2:
            return $instance->$method($args[0], $args[1]);
        case 3:
            return $instance->$method($args[0], $args[1], $args[2]);
        case 4:
            return $instance->$method($args[0], $args[1], $args[2], $args[3]);
        default:
            return call_user_func_array([$instance, $method], $args);
    }
}
```

然后我们看 `__callStatic()` 的执行过程。首先看 `getFacadeRoot()` 是如何执行的

```php
public static function getFacadeRoot()  
{
    return static::resolveFacadeInstance(static::getFacadeAccessor());
}
```

在最初，定义 Route 类时，我们只实现了一个方法 `getFacadeAccessor()`，这时我们当初定义的字符串，就会在此处用到了，所以上面这个函数，实际上返回的内容就是 `static::resolveFacadeInstance("router");`

我们继续看 `resolveFacadeInstance` 这个函数的执行过程

```php
protected static function resolveFacadeInstance($name)
{   
    //判断是否为对象，当然不是了，$name 是字符串
    if (is_object($name)) {
        return $name;
    }
    
    //判断 resolvedInstance 这个数组中是否存了 $name 相关的信息，当然也没有，因为我们假设程序是第一次执行这里
    if (isset(static::$resolvedInstance[$name])) {
        return static::$resolvedInstance[$name];
    }
    
    // 返回 static::$app[$name]，同时把得到的结果保存到上面验证的数组中
    return static::$resolvedInstance[$name] = static::$app[$name];
}
```

所以我们的程序执行了最后的一个 return， 返回了 static::$app['router'] 这个值。
还记得 $app 吗，他就是前面说过的 Laravel  Application 类的实例化对象，这个类是一个 IOC Container，实例化过程发生在  Laravel  最开始的时候。

在 Facade 初始化的时候，也让自己有了一个 `static::$app` 这个就是 Application 类的实例化对象

而 $app 其实并没有 'router' 这个属性，那为什么可以这样调用呢？ 是因为 Application 继承了 Container， 而 Container 又继承了 ArrayAccess 这个类。正是由于 ArrayAccess 的存在，以及 Container 实现了 ArrayAccess 的下面这个方法：

```php
public function offsetGet($key)
{
  return $this->make($key);
}
```

所以，当我们使用 $app['router'] 时，实际上是执行了 $app->make('router')，到这里已经比较明显了，这里就是从 $app 这个 IOC Container 中，`make` 了一个 router 的实例。

已经绕的有点远了，不过还好，我们终于要回去了。到最初的 `__callStatic()` 中的 

$instance = static::getFacadeRoot();

也就相当于 

$instance = $app->make('router')

`__callStatic()` 继续往下执行，想一下我们最初的那条代码 

`Route::get('/', 'HomeController@index');`， 有2个参数，所以会执行到 case 2 这条语句，

```php
case 2:
            return $instance->$method($args[0], $args[1]);
```

到这里，我们的 $instance 就是我们的 IOC Container `make` 出的具有实际功能的实例，这个实例将会执行这个实例的类所声明过的 `get` 方法，并使用这两个参数：'/' 和 'HomeController@index'。

##### 如何将某个类通过 Service Provider 的方式， `bind` 到 Laravel  的 IOC Container 中？

上面通过 `Route::get('/', 'HomeController@index');` 这行代码背后的故事，让我们知道了 Facade 是用来帮我们从 IOC Container 中 `make` 实例的。

文章一开始就讲了，既然你要 `make`， 必定要先 `bind`。
上面还讲过，为什么通常情况下都是通过 Service Provider 来 `bind`。

那么我们现在就还是以 Route 为例子，来看看  Laravel  的开发者是如何通过 Service Provider 来 `bind` 类的。

在之前的 基础版 IOC Container 中, 我们看到 不论是 `bind` 还是 `make` 都有一个 key，用来查找和保存我们 `bind` 过的类。
上文说过，代码实际执行了  $app->make('router')，那显然，这个 key 此时就是 'router'。我们可以肯定，在之前进行 `bind` 操作的时候，也一定用到了这个个字符串 'router'。

[官方文档](https://laravel.com/docs/5.3/providers)的 Service Provider 这一章中描述了 如何注册一个 Service Provider（这里就不做过多介绍），

以及所有的 Service Provider 都在 `config/app.php` 中被注册，

在 `config/app.php` 中，很容易就能找到跟我们的 Route 相关的， 也就是 `App\Providers\RoutingServiceProvider::class,`

按照这个路径我们找到这个 Service Provider

```php
class RoutingServiceProvider extends ServiceProvider
{
    /**
     * Register the service provider.
     *
     * @return void
     */
    public function register()
    {
        $this->registerRouter();

        $this->registerUrlGenerator();

        $this->registerRedirector();

        $this->registerPsrRequest();

        $this->registerPsrResponse();

        $this->registerResponseFactory();
    }

    /**
     * Register the router instance.
     *
     * @return void
     */
    protected function registerRouter()
    {
        $this->app['router'] = $this->app->share(function ($app) {
            return new Router($app['events'], $app);
        });
    }
```

就像我们预计的那样，和官方文档中说的一样，在这个 Serivce Provider 的 `register()` 方法中，完成了 `bind` 的动作, 而 `make` 实例的方法也写在了里面， 也就是

`new Router`， 这个 Router 类去提供实际的功能。

既然说到 Router 类了，那就不得不提一下我们的4号角色 Contract。

Router 类是为他人提供服务的功能类，比如说上文的 `get` 方法，这就是一个功能，看一下 Router 类的代码

```php
namespace Illuminate\Routing;

use Illuminate\Contracts\Routing\Registrar as RegistrarContract;

class Router implements RegistrarContract
{
    use Macroable;
```

可以看到 Router 类实现了 RegistrarContract 这个接口，这个接口的命名空间位于 Illuminate\Contracts 之下，而这种接口在  Laravel  中就被称为 Contrast。


```php
<?php

namespace Illuminate\Contracts\Routing;

use Closure;

interface Registrar

```

那这种接口有啥好处呢， 跟普通的 interface 有什么不同？

答案可能让大家失望了，并没有什么不同。那为什么要叫俩名呢，我觉得你可以这样理解：

interface 这个词，在编程领域的有太广泛的应用了。但在  Laravel  框架中，特别是在框架的 Illuminate\Contracts 这个命名空间下的这些  Laravel  框架自带的接口们，我们把这些接口特指为 Contrast。



</br></br></br></br>


## 总结

这篇文章写了蛮久，资料也看了很多，源代码也翻了不少，好在最终总算能让我感觉把这件事情的几个重要的点都描述清楚。

但是还有一些遗憾，比如在最初写文章的时候还希望能让大家了解 「如何写一个自己的 Service Provider」，但无奈本文篇幅已经太长，而且这部分内容不是太难，网上相关资料也比较多，就不再在本文中详述了。

再比如 「如何写一个自己的 Facade」。

以后可能会再添加吧。

多谢能为我提供资料的作者们。

大家对本文有什么看法和问题都欢迎评论，有错误也非常感谢你能帮我指出。毕竟能帮到别人才是我写这片文章的最重要的目的。还有一个小小的目的，最近在找一份  Laravel  的实习或者初级开发的工作，怕面试官问我 Laravel 的几个核心概念我答不上来。

</br>
## 后续

Facade 不仅仅是  Laravel  框架中的一个类，而且 Facade 是一种设计模式。

维基百科是这样描述的：
>The Facade design pattern is often used when a system is very complex or difficult to understand because the system has a large number of interdependent classes or its source code is unavailable. This pattern hides the complexities of the larger system and provides a simpler interface to the client.     

我翻译过后大概就是：
当一个系统过于复杂，庞大和难以理解时，Facade 设计模式就经常会被使用到。难以理解的原因通常是由于这些系统中存在大量相互依赖的类，而这些类的源代码有时候你是无法找到的。Facade 设计模式可以帮助庞大的系统隐藏其背后的复杂性，并且提供了一个更加清晰而简单的给需要这个系统的客户端使用。



## 想要知道更多
感谢以下的作者们提供思路。

[Laravel 学习笔记 —— 神奇的服务容器](https://laravel-china.org/topics/789)

[Stackoverflow 上的问答：Differences between contracts and facades laravel?  ](http://stackoverflow.com/questions/33873675/differences-between-contracts-and-facades-laravel)

[Laravel-Facade 实现原理](http://ju.outofmemory.cn/entry/136407)

[想对 Contracts 了解更多可以看这里：Laravel 之 Contracts 和 Facades](https://segmentfault.com/a/1190000004946198)

[Laravel 5.1 文档攻略 ——Facades](http://larabase.com/collection/1/post/49)

[Laravel 核心要点的个人理解帮助](http://blog.11010.net/laravel-help-understand)

[vimeo 上的视频教程：Laravel Architecture - Part 1: The IoC Container](https://vimeo.com/88888213)

[ Laravel  官方文档 ](https://laravel.com/docs/5.3)
