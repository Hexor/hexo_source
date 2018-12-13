---
title: Hexo 常用使用手册
date: 2016-11-27 08:26:21
tags: ['Hexo']
categories: 技术
---
<img src="/images/rivergod.png" width=50% height=50% align=center/>
本文将介绍我使用 Hexo 时的一些常用操作

<!--more-->
<!-- toc -->

<br>
## 文章内容

#### 生成大纲
``` markdown
<!-- toc -->
```


#### 首页上全文/预览显示 分割
``` markdown
<!--more-->
```


#### 代码添加
``` markdown

<!-- 将 ^ 替换成 ` -->

^^^ php
^^^

```


#### 添加图片

``` html
<!-- 图片放到 ../hexo/source/images/ 文件夹下 -->
<img src="/images/rivergod.png" width=50% height=50% align=center/>

```



<br>
## 文章基本信息

#### 设置标签
``` markdown
    tags: ['Hexo','Php',...]
```

#### 设置分类
``` markdown
    categories: 技术
```

#### 设置文章图片
``` markdown
    photos: 
    - /images/rivergod.png
```

#### 比如本文的基本信息为
``` markdown
    title: Hexo 常用使用手册
    date: 2016-11-27 08:26:21
    tags: ['Hexo']
    categories: 技术
    photos: 
    - /images/rivergod.png
```

#### 设置访问次数

[不蒜子服务](http://ibruce.info/2015/04/04/busuanzi/)


#### hexo 文章置顶 设置

编辑 hexo 目录下的 `node_modules/hexo-generator-index/lib/generator.js` 文件

将其修改为
``` javascript

'use strict';
var pagination = require('hexo-pagination');
module.exports = function(locals){
  var config = this.config;
  var posts = locals.posts;
    posts.data = posts.data.sort(function(a, b) {
        if(a.top && b.top) { // 两篇文章top都有定义
            if(a.top == b.top) return b.date - a.date; // 若top值一样则按照文章日期降序排
            else return b.top - a.top; // 否则按照top值降序排
        }
        else if(a.top && !b.top) { // 以下是只有一篇文章top有定义，那么将有top的排在前面（这里用异或操作居然不行233）
            return -1;
        }
        else if(!a.top && b.top) {
            return 1;
        }
        else return b.date - a.date; // 都没定义按照文章日期降序排
    });
  var paginationDir = config.pagination_dir || 'page';
  return pagination('', posts, {
    perPage: config.index_generator.per_page,
    layout: ['index', 'archive'],
    format: paginationDir + '/%d/',
    data: {
      __index: true
    }
  });
};
```

然后在每篇文章的顶部，即
```
...
title: Hexo 常用使用手册
date: 2016-11-27 08:26:21
...
```
在这个位置，加入 `top` 字段，top 值最大的文章，就会被置顶

```
...
title: Hexo 常用使用手册
date: 2016-11-27 08:26:21
top: 10
...
```



<br>


## 想知道更多

[Hexo使用攻略：（四）Hexo的分类和标签设置](http://ijiaober.github.io/2014/08/05/hexo/hexo-04/)
[How to creat a "galleries page" in hexo?](https://github.com/hexojs/hexo/issues/912)
[解决Hexo置顶问题](http://www.netcan666.com/2015/11/22/%E8%A7%A3%E5%86%B3Hexo%E7%BD%AE%E9%A1%B6%E9%97%AE%E9%A2%98/)



    