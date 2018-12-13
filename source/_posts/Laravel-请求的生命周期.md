---
title: Laravel 请求的生命周期
date: 2016-11-30 21:25:09
tags: ['Laravel']
categories: 技术
photos: 
- /images/laravel_lifecycle_overview.png
---

当你使用一个工具的时候, 如果你对这个工具的内部原理和构造有所了解, 那么在使用这个工具的时候, 就会更加的有信心, 工具用起来也会更加的得心应手. 

<!--more-->
<!-- toc -->

今天阅读了 Laravel 官方的[生命周期文档](https://laravel.com/docs/5.2/lifecycle). 这篇文章可以看做是对官方文档的翻译, 但是也加入了当前我对 Laravel 请求生命周期的理解, 同时也算是加深我对 Laravel 整体架构的印象.

## First Things

首先, 从请求说起, 所有的来自于 web 服务器 ( Apache / Nginx ) 的请求, 都将会被转发到 `public/index.php` 这个文件处理.

`index.php` 中代码不多, 但这个文件是一个起始点, 从这个文件开始, Laravel 框架的其余部分就会开始陆续加载了.
同时, `index.php` 从 `bootstrap/app.php` 获得了一个实例, 而这也是 Laravel 框架在得到请求之后做的第一件事.

</br>

## HTTP / Console Kernels

通过之前得到的这个实例, Laravel 便可以生成处理请求的内核了.
Laravel 本身提供2种内核, HTTP 内核和 Console 内核, 2种内核分别处理不同类型的请求. 我理解的是 HTTP 内核可能是用来处理 HTTP 请求, 而 Console 内核则用来处理控制台中发送来的请求. 文章中暂时只让我们关注于 HTTP 内核, 可能这也是最常用的.

HTTP 内核文件位于 `app/Http/Kernel.php`, 继承自 `Illuminate\Foundation\Http\Kernel` Kernel 这个 Class.

在 `Illuminate\Foundation\Http\Kernel` Kernel Class 中, 定义了一个由 `bootstrappers` 组成的数组, 会有一个函数遍历数组中的每个 `bootstrapper`, 并执行每一个 `bootstrapper`. 这些 `bootstrappers` 在请求被真正的业务逻辑处理之前, 会执行一些包括错误处理, 日志记录, 确定应用环境等配置任务.

HTTP 内核还定义了一个中间件的列表 ( a list of HTTP middleware ), 所有的请求在被处理之前, 都需要先通过这些中间件的处理. 这些中间件可以处理 HTTP session 的读写, 可以判断服务器当前是否处于维护模式, 验证 CSRF token ( 为了保护服务器不受 [CSRF 攻击](https://laravel.com/docs/5.2/routing#csrf-protection) ) 等等功能.

最后, HTTP 内核最终用于处理请求的方法 ( `handle` method) 看起来还是很简单的. 单纯的接受一个 `Request` 以及 返回一个 `Response`.

上面这些概念有的地方也许我现在还无法详细的理解, 想要充分的理解, 就需要去看 Laravel 的源码了. 但是目前这个阶段可以先不需要看源码. 
只需要知道, 内核就是一个黑箱, 可以为你提供服务器的所有功能, 而你只需要传给内核 HTTP 请求, 黑箱就会返回 HTTP 响应了.

### Service Providers

在 Kernel Class `bootstrappers` 数组中, 最重要的 `bootstrapper` 任务就是载入 `service provider`, 这项任务由 RegisterProviders 和 BootProviders 2个 `bootstrapper` 来完成.

所有的 service provider 都会在 `config/app.php` 中的 `providers` 中进行配置. 这些 'providers' 的启动方式是: 首先所有的 provider 都会执行 register 方法, 一旦所有的 provider 都执行完毕 register 方法, `boot` 方法就会调用.

Service Provider 的作用非常重要, 它会启动框架的所有组件, 包括数据库, 队列, 数据验证, 路由组件等. 也正是由于 Service Provider 启动和配置了所有 Laravel 框架的特性和功能, 所以才让它成为了整个 Laravel 启动过程中最终要的部分.


### Dispatch Request

一旦所有的 `bootstrapper` 都执行完, 所有的 Service Provider 都注册完之后, 请求终于到达了 `router`, `router` 将会把得到的请求分发给其下一级 ( 通常是一个 controller 或者另一个 router ), 当然如果设置任何 router 相关的 中间件, 中间件也会先执行.

</br>

## Focus On Service Providers

官方文档说, 大家一定要把注意力放到 Service Providers 上啊. 毕竟整个 Laravel 的工作流程简单来说就是, 创建一个实例, 注册所有的 Service Provider, 请求就可以开始被处理了. 

系统的默认 Service Providers 放在 `app/Providers` 目录下. 默认情况下, `AppServiceProvider` 是一个比较空的文件, 如果你想要放置自定义的 初始化任务 或者 绑定一个 service container, 这里就是做这些事情的合适位置. 

但是如果对一个大型的项目来说, 创建多个你自己的 Service Provider 也许更好, 从而在每个 Service Provider 中进行粒度更细分的初始化任务.

</br>


## 想知道更多

[Laravel 官方文档: Request Lifecycle](https://laravel.com/docs/5.2/lifecycle)
[Laravel 学习笔记之 Bootstrap 源码解析](https://laravel-china.org/topics/2909)