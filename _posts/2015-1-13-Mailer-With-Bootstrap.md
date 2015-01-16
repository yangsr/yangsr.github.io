---
layout: post
title: Rails Mailer View层整合Bootstrap实践
---

### Email的局限性
最近平台发出去的Email界面频频被用户吐槽，”完全就是纯文本“。当初写mailer功能时，曾乐观地以为可以直接复用已有的view页面，这样也便于以后的维护。结果发现：原来Email的格式相对于HTML有诸多局限：

* 无法链接外部CSS、Fonts
* 无法使用`<head>`标签
* Javascript被限制
* 部分CSS属性的跨平台跨邮件服务商的兼容性问题：比如margin在Hotmail中就被忽略

既然限制这么多，那应该怎样设置Email的样式呢？答案是使用**内联样式**，比如：
{% highlight html %}
<h1 style="color: red;">Hello</h1>
{% endhighlight %}

### Premailer
将Email页面中的每部分都加上内联样式是一项非常枯燥的工作，而且不便于维护，好在一个叫[Premailer](https://github.com/fphilipe/premailer-rails)的Gem包可以很好地完成这方面工作。

只需要将
{% highlight ruby %}
gem 'nokogiri'
gem 'premailer-rails'
{% endhighlight %}
添加到你的Gemfile里，然后bundle一下就可以完成安装了。

这样，当你发送一封邮件的时候，Premailer会以一定的顺序(Cache,File System,Asset Pipeline,Network)去搜集CSS，并将它们嵌入Email页面中HTML元素的style属性中。

### 整合Bootstrap
我们平台的前端使用了Bootstrap-sass，为了在email中实现bootstrap中一些组件的效果，在app/assets/stylesheets目录下新建email.css.scss文件，内容如下：
{% highlight scss %}
@import "bootstrap-sprockets";
@import "bootstrap/variables";
@import "bootstrap/mixins";
@import "bootstrap/scaffolding";
@import "bootstrap/type";
@import "bootstrap/buttons";
@import "bootstrap/alerts";
@import 'bootstrap/normalize';
@import 'bootstrap/tables';
@import 'bootstrap/progress-bars';
{% endhighlight %}

然后，在Mailer的View层的`<head>`标签中，加入：
{% highlight erb %}
<%= stylesheet_link_tag "email" %>
{% endhighlight %}
这样，一些Bootstrap的组件就可以在Email中显示了。

### 测试
前面已经说过，Email中会遇到一些CSS属性在不同平台不同邮件服务商下的兼容性问题，当你引入Bootstrap后，同样也需要仔细测试。

你需要：

* 测试你的邮件在IE/Chrome/Firefox等主流浏览器下的显示效果
* 测试你的邮件在不同邮件服务提供商(Gmail/网易邮箱/qq邮箱)下的显示效果
* 测试你的邮件在Outlook/Thunderbird/网易闪电邮等邮件接收客户端上的显示效果
* 如果考虑到用户可能会使用手机接收邮件，你同样需要进行相应的测试

[这里](https://litmus.com/pub/d831ecc/screenshots)有一些别人已经测试过的截图，供参考。