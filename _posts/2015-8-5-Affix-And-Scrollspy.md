---
layout: post
title: 导航栏悬浮定位效果的实现思路
---

### 需求说明
最近网站改版，需要实现一个悬浮定位导航栏，类似效果见下图（[Bootstrap页面示例](http://v3.bootcss.com/components/)）：

![Bootstrap悬浮定位导航示例]({{ site.baseurl }}/images/demo.png "Bootstrap悬浮定位导航示例")

本篇博文将主要讲述该效果的具体实现细节，实现依赖于Bootstrap框架。

### 搭建框架
首先，整个html页面的基础框架结构是这样的：

```html
<div class="col-md-3">
  <ul id="nav" class="nav hidden-xs hidden-sm">
    <li>
      <a href="#web-design">Web Design</a>
    </li>
    <li>
      <a href="#web-development">Web Development</a>
      <ul class="nav">
        <li>
          <a href="#ruby">
            <span class="fa fa-angle-double-right"></span>Ruby
          </a>
        </li>
        <li>
          <a href="#python">
            <span class="fa fa-angle-double-right"></span>Python
          </a>
        </li>
      </ul><!--end of sub navigation-->
    </li>
  </ul><!-- end of main navigation -->
</div>
<div class="col-md-9">
  <section id="web-design">
  </section>
  <section id="web-development">
    <section id="ruby">
    </section>
    <section id="python">
    </section>
  </section>
</div>
```
我们将页面划分为3:9的两部分，左侧为导航栏，右侧为具体内容。

在这个例子中，我们希望当屏幕缩小到一定比例时，悬浮定位导航栏自动隐藏，因此我们为导航添加了`hidden-xs`和`hidden-sm`类。

每一个`<section>`都有一个id，对应着`<a>`标签中的href属性，即锚点。

### Affix控件
Bootstrap的Affix控件在这里的主要功能是实现当页面向下滑动时导航栏的悬停效果，要使用Affix控件很容易，直接将`data-spy="affix"`添加到导航栏上就好。

```html
<ul id="nav" class="nav hidden-xs hidden-sm" data-spy="affix" data-offset-top="380" >
```
注意上图中的`data-offset-top`属性，它设置的是导航栏距离顶部多少像素时开始悬停。

下面稍稍解释下本例中Affix控件的工作流程：

1.初始状态下，我们可以注意到控件将`affix-top`添加导航栏的类中：

![affix-top示例]({{ site.baseurl }}/images/affix_top.png "affix-top示例")

2.当页面滚动超过我们设定的像素距离后，控件用`affix`替换了`affix-top`：

![affix示例]({{ site.baseurl }}/images/affix.png "affix示例")

这时我们可以通过设置css样式调整导航栏悬停时的效果：

```css
.affix {
  top: 20px;
  width: 213px;
}
```

### ScrollSpy控件
Bootstrap的滚动监听ScrollSpy控件在这里的作用是当页面主体内容改变时，导航条也要有对应的变化（子章节的展开、高亮等）。

ScrollSpy控件依赖Bootstrap的[导航控件](http://v3.bootcss.com/components/#nav)用于高亮当前激活的链接。

同时，需要被监听的组件是`position: relative;`，即相对定位方式，大多时候是监听body元素。

我们可以通过data属性调用ScrollSpy控件，将`data-spy="scroll"`添加到你希望监听的元素上（`<body>`），同时将`data-target`设置为本例中导航栏的class：

```html
<body data-spy="scroll" data-target=".scrollspy">
<!-- content here... -->

  <div class="col-md-3 scrollspy">
    <ul id="nav" class="nav hidden-xs hidden-sm" data-spy="affix">
        <!-- nav items here... -->
    </ul>
  </div>

  <!-- content here... -->
</body>
```
记得设置`<body>`元素的CSS:

```css

body {
  position: relative;
}
```
当用户滑动鼠标滚轮，页面内容发生变化时，我们可以观察到ScrollSpy控件将`active`添加到对应导航栏中的`<li>`元素的类中：

![ScrollSpy示例]({{ site.baseurl }}/images/scroll_spy.png "ScrollSpy示例")

接下来要做的就是为导航栏的各种状态添加一些css效果即可：

```css
.nav .active {
  font-weight: bold;
  background: #72bcd4;
}

.nav .nav {
  display: none;
}

.nav .active .nav {
  display: block;
}

.nav .nav a {
  font-weight: normal;
  font-size: .85em;
}

.nav .nav span {
  margin: 0 5px 0 2px;
}

.nav .nav .active a,
.nav .nav .active:hover a,
.nav .nav .active:focus a {
  font-weight: bold;
  padding-left: 30px;
  border-left: 5px solid black;
}

.nav .nav .active span,
.nav .nav .active:hover span,
.nav .nav .active:focus span {
  display: none;
}
```
### 参考

[Understanding Bootstrap’s Affix and ScrollSpy plugins](http://www.sitepoint.com/understanding-bootstraps-affix-scrollspy-plugins/)