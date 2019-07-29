# AwsDemoCenter.github.io

## 1.jekyll 简介
jekyll 是一个静态网站生成器.通过 标记语言 markdown 或 textile 和 模板引擎 liquid 转换生成网页.jekyll只是一个生成静态网页的工具，不需要数据库支持。最关键的是jekyll可以免费部署在Github上，而且可以绑定自己的域名。


## 2.安装jekyll
安装的目的是为了能够实现本地预览，查看我们修改的网站样式  
安装参考：https://www.jianshu.com/p/9f71e260925d

## 3.安装过程中出现错误
初次安装出现错误，大部分是因为所需要的依赖包没有安装到位，这时候我们需要手动安装报错的安装包(gem install package)；如果安装包版本升级，即使你安装了高版本的安装包也会报错，这时需要在gemefile.lock中更改安装包的版本

出现错误参考：http://szhshp.org/tech/2015/11/14/localjekyllenv.html

## 4.jekyll模板的介绍
### _config.yml
jekyll 的全局配置在 _config.yml 文件中配置.
比如网站的名字, 网站的域名, 网站的链接格式等等.
### _includes
对于网站的头部, 底部, 侧栏等公共部分, 为了维护方便, 可以提取出来单独编写, 然后使用的时候包含进去即可.
这时我们可以把那些公共部分放在这个目录下.
使用时只需要引入即可 { % include filename % }
### _layouts
 对于网站的布局,我们一般会写成模板的形式,这样对于写实质性的内容时,比如文章,只需要专心写文章的内容, 然后加个标签指定用哪个模板即可.
对于内容,指定模板后,我们可以称内容是模板的儿子.只需要：
在模板中, 引入儿子的内容.{ { content } }
在儿子中,指定父节点模板。注意,必须在子节点的顶部.
```
---
layout: post
---
```

### _posts
里面存放的文件是我们编写的网站内容,文件后缀以 .mardown结尾，在文件的头部必须指定布局以及title
```
---
layout:  post
title: 
---
```

### _site
jekyll 生成网站输出的地方, 一般需要在 .gitignore 中屏蔽掉这个目录.

### index.html
主页文件, 后缀有时也用 index.md 等.
这个需要根据自己的需要来写, 因为不同的格式之间在某些情况下还是有一些细微的差别的.


### 静态资源
对于其他静态资源, 可以直接放在根目录或任何其他目录, 按路径来找链接中的文件.

### .gitignore
忽略其中的文件或文件夹，这个很有用, 有时候你写了一个文件, 里面的一个东西可能会被 jekyll 处理, 但是你不想让 jekyll 处理的话, 就使用这个语法忽略那些文件吧.
参考网址：http://github.tiankonguse.com/blog/2014/11/10/jekyll-study.html


## 5.我们的网站：
网站地址：https://awsdemo.github.io/  
github地址：https://github.com/AwsDemo/AwsDemo.github.io

## 6.demo的编写：
如果需要新增一个demo，只需要在_post文件夹下加一个 .markdown文件，文件的标题为 year-month-day-title (xxxx-xx-xx-title)，然后在文章的开头
指定布局以及title即可
```
---
layout:  post
title: 
toc: true
---
```
如果需要增加图片，请使用以下方式,可以支持图片的放大预览：
```
<a data-fancybox="gallery" href=" ">
![]()
```
