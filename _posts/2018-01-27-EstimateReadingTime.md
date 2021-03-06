---
layout: post
title: 不使用任何插件,也能让中文博客文章显示"阅读需要的时间"
date: 2018-1-28
categories: jekyll
tags: [jekyll]
excerpt: 使用Jekyll模板在github pages上搭建自己的博客很方便，但是貌似很多模板都不支持在博客上显示“阅读需要的时间”。那有什么简单方法可以给自己的博客添加这个功能呢？中文博客又有什么特殊情况要处理？
---

最近尝试着在github pages上套用大牛们的模板搭建了自己的技术博客，感觉很好玩。以前很羡慕别人的博客如何酷炫，如今在github上看到谁的博客风格比较对自己的胃口, clone一下把Repository改成自己的username.github.io,在修改一下个人信息，自己的博客马上就大变身了，简直不能再任性。但我偶尔发现有些博客上可以自动显示“阅读需要的时间“，而我clone的博客模板上却没有这个贴心的小功能，怎么才能把它移植过来呢？

#### 基本原理探究

简单一搜索，就找到了这篇文章[Jekyll: Reading time without plugins](https://carlosbecker.com/posts/jekyll-reading-time-without-plugins/)，原理很简单:

1.  统计博客文字量
2.  设定一个经验阅读速度，计算出阅读时间
3.  在博客上显示计算出的时间

代码也很简单，即使没有接触过html/css语言，理解起来也没有什么困难。
步骤1,2的代码可以放在一个单独的reading_time.html文件里，内容如下:
```html
{% raw %}
<span class="reading-time" title="Estimated read time">
  {% assign words = content | number_of_words %}
  {% if words < 360 %}
    1 min
  {% else %}
    {{ words | divided_by:180 }} mins
  {% endif %}
</span>
{% endraw %}
```
找到自己博客的layout模板文件后，步骤3一行代码就能搞定:
```html
{% raw %}
{% include reading_time.html %}
{% endraw %}
```

#### 方案移植问题多多

纸上得来终觉浅，缘知此事要躬行。把上述代码移植到我的博客模板中试试看吧。

我的博客模板fork自[HyG's Blog](https://github.com/Gaohaoyang/gaohaoyang.github.io)(感谢原博主[Gaohaoyang](https://github.com/Gaohaoyang)), Jekyll模板公共的html文件都放在\_includes文件夹，所以我把reading_time.html也放到了这个文件夹里。因为经常会在博客上贴出代码,所以我把阅读速度从360字/秒调低了一些到260字/秒;然后在\_layout文件内，找到博客正文layout的模板文件post.html,在tag之后插入reading_time.html:

```html
{% raw %}
<div class="label-card">
	{% include reading_time.html %}
</div> 
{% endraw %}
```

我的本机没有安装Jekyll环境，所以直接push移植代码到github. 再次点开我的博客，发现每篇博客的tag右边都显示出了 "1 min",移植一次成功！

##### 准确统计中文字数统计

但是仔细一看,我有几篇上千字的中文博客，所需的阅读时间怎么也会是"1 min"? 再看一下reading_time.html的代码，输出*1 min*的条件是*words < 260*, 难道是中文字数统计不准确？不管是网络应用还是传统Desktop软件，国外开发者对CJK的支持一向不敢恭维，也许这次碰到同样的问题了？ Google一下自己的怀疑，还真被自己猜中了:

[The filter number_of_words can't parse Chinese correctly](https://github.com/jekyll/jekyll/issues/1921)。

Jekyll内建支持的filter 'number_of_words'不能准确统计中文字数，好在可以使用Liquid 的'size' filter绕开这个问题。为了统计更精确，在计算之前最好把所有html标记和空行也忽略掉:

```html
{% raw %}
{% assign words = content | strip_html | strip_newlines | split: "" | size %}
{% endraw %}
```

##### 支持博客正文页和索引页

再次push代码到github,这次中文博客正文所需的阅读时间果然可以准确显示了，但是我的博客首页是一个按发布时间显示正文摘要的索引页，怎么才能把reading_time也显示出在索引页上呢？

好在我fork的索引模板已经支持给每篇摘要显示category和tag了，参考一下/_includes/tag.html和/_includes/category.html模板文件，原来可以通过判断*post*变量是否存在来区分page和post。那么对reading_time.html做如下改动，就可以同时支持索引页和博客正文页了:

```html
{% raw %}
{% if post %}
	{% assign words = post.content | strip_html | strip_newlines | split: "" | size %}
{% else %}
	{% assign words = page.content | strip_html | strip_newlines | split: "" | size %}
{% endif %}
{% endraw %}
```

然后借鉴category和tag的显示代码，相应地在/index.html模板文件中include reading_time.html,这样就能在博客首页的博文摘要上方，看到正文所需的阅读时间了:

```html
{% raw %}
<div class="label-card">
    {% include reading_time.html %}
</div>
{% endraw %}
```

##### 添加时钟图标

再仔细看看这个小功能,怎么感觉还是有什么地方怪怪的?对了,category和tag都有图标,reading_time也应该有图标风格才能保持一致呀。 怎么添加图标呢？

* 直接嵌入svg

Google学习了一下，现代浏览器都已经支持svg图片格式了，所以最简单的办法就是找一张svg格式的时钟图标文件，复制svg文件内容，直接用svg标签内嵌到html中。svg矢量图片可以无损缩放，所以不用担心图片的尺寸大小，在svg标签中指定宽高16*16,图标自动就缩放合适了。代码如下:

```html
<svg id="i-clock" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 32 32" width="16" height="16" fill="none" stroke="currentcolor" stroke-linecap="round" stroke-linejoin="round" stroke-width="2">
    <circle cx="16" cy="16" r="14" />
    <path d="M16 8 L16 16 20 20" />
</svg>
```

* 嵌入base64编码的svg图片

茴香豆“茴”字的另一种写法是使用base64编码的图片，这样别人即使查看你的html源码也只能看到一串火星文，搜索引擎也拿它没办法,高冷！关于怎么玩转base64编码的图片，感兴趣的同学可以参考[Base64 encoding images](https://varvy.com/pagespeed/base64-images.html)。把上述svg图片base64加密之后, 用*img*标签就可以将其嵌入到html,达到同样的效果:

```html
<img height="16" width="16" src="data:image/svg+xml;base64,PHN2ZyBpZD0iaS1jbG9jayIgeG1sbnM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIiB2aWV3Qm94PSIwIDAgMzIgMzIiIHdpZHRoPSIzMiIgaGVpZ2h0PSIzMiIgZmlsbD0ibm9uZSIgc3Ryb2tlPSJjdXJyZW50Y29sb3IiIHN0cm9rZS1saW5lY2FwPSJyb3VuZCIgc3Ryb2tlLWxpbmVqb2luPSJyb3VuZCIgc3Ryb2tlLXdpZHRoPSIyIj4KICAgIDxjaXJjbGUgY3g9IjE2IiBjeT0iMTYiIHI9IjE0IiAvPgogICAgPHBhdGggZD0iTTE2IDggTDE2IDE2IDIwIDIwIiAvPgo8L3N2Zz4=" />
```
* 使用Font Awesome图标

作为html小白，我刚为找到了两种“茴”字的写法沾沾自喜没多久，就发现原有的tag.html和categories.html文件并没有使用我找到的两种方法，而是用以下标记显示图标：
```html
<span class="pageTag">
  <i class="fa fa-tags"></i>{{ tags }}
</span>
```
fa是什么东东？fa-tags在哪里？为什么我在本地搜索不到，也没有看到引用链接？搜索了解了一下html `<i>`标记，原来fa代表的是Font Awesome icons,只需要把下面几行代码放在html `<head>`内,不需要下载和安装，就可以用上述`<span>`和`<i>`标记在html中添加图标，真是再方便不过了。
```html
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/4.7.0/css/font-awesome.min.css">
```
回过头又看了一下我博客模板的\_includes/head.html文件,果然发现了对font-awesome的引用。看来对html/css不了解如我者，尝试自定义github pages免不了会走很多弯路呀。
#### 最终效果

最后，贴上我的博客截图效果:
![mins to read](/img/posts/readingtime/blogsnap.png)
如果想获取完整源代码,请移步[reading_time](https://raw.githubusercontent.com/egoechog/egoechog.GitHub.io/master/_includes/reading_time.html)。
