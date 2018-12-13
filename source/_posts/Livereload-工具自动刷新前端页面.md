---
title: Livereload 工具自动刷新前端页面
date: 2016-12-16 08:20:01
tags: ['Laravel']
categories: 技术
---

在开发中，经常遇到这种情景：修改项目文件后，要去浏览器中刷新一下页面，检查最新的修改结果。这里介绍一个小工具，他可以在你修改特定的代码之后，自动为你刷新浏览器开发页面。

<!--more-->
<!-- toc -->



本教程使用的版本为  Laravel 5.1，其他  Laravel 5.* 版本应该也适用。
Windows 环境下没有进行过配置，不保证成功。

## 修改 gulpfile

在  Laravel  项目文件中找到 gulpfile.js，在文件末尾加入以下代码，以下示例代码监听的是 *.blade.php 类型的文件修改动作，如需监听其他文件，作出类似的修改即可。


```
var gulp = require('gulp');
var livereload = require('gulp-livereload');

/**
 * task - 'default'
 * executes 'live-monitor'
 */
gulp.task('default', ['live-monitor']);

/**
 * task - 'laravel-views'
 * monitor laravel views
 */
gulp.task('laravel-views', function() {
    gulp.src('resources/views/**/*.blade.php')
        .pipe(livereload());
});

/**
 * task - 'live-monitor'
 * monitors everything
 */
gulp.task('live-monitor', function() {
    livereload.listen();
    gulp.watch('resources/views/**/*.blade.php', ['laravel-views']);
});

```


## 安装 gulp-livereload

首先，我们在项目的路径下，运行

```
npm install
```

完成之后，我们试着运行 gulp 

发现会报错 
Error: Cannot find module 'gulp-livereload' 

此时，我们用下面一行安装 livereload，运行

```
npm install --save-dev gulp-livereload
```


## 运行 gulp
直接在项目路径下运行 `gulp` 命令即可，运行成功之后，会有类似下面一样的打印显示出来

```
Using gulpfile ~/Workspace/PHP/myTodoPractice/gulpfile.js
Starting 'live-monitor'...
Finished 'live-monitor' after 24 ms
Starting 'default'...
Finished 'default' after 13 μs
```


## 安装 Chrome 插件

插件地址：[Chrome 插件商店](https://chrome.google.com/webstore/detail/livereload/jnihajbhpnppcggbcgedagnkighmdlei)

安装完成之后，在 Chrome 中打开我们想要自动刷新的网页（比如说 localhost 或者 homestead.app），左键单击插件图标。
此时，插件图标中心的圆圈符号会从空心变为实心，则说明插件已经开始与 gulp-livereload 配合工作了。


## 使用方法

以上都配置之后，当你使用编辑器对符合之前的监听条件的文件作出修改，并 `save` 之后，Chrome 插件便会帮你自动刷新你要调试的网页了。

