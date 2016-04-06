---
layout: post
title: 使用will_paginate实现下拉滚动异步加载效果
---

### 由交互变更聊起
我们网站的结果页面可能有上百条数据，原本采用的是传统的分页模式，即点击“上一页”或“下一页”按钮进行翻页操作。最近进行了一次交互改版，新版的交互变更为当鼠标滚轮滚动到页面底部时，页面上自动加载出下一页的内容。这篇博客要介绍的就是怎样实现这种交互效果。

### 实现思路概述
在传统的分页模式下，当我们点击“下一页”按钮时，实际上是向后端发送了一个AJAX请求，在这个请求的URL中附带了参数`page`的信息，后端接收到请求后，根据`page`参数利用`will_paginate`取出分页后的数据，最后更新前端页面。

在新版交互模式下，其实只需要在原有的分页功能基础上做一个小更改：当鼠标滚轮滚动到页面底部时，向后端发送一个AJAX请求，请求的URL即传统的“下一页”的链接URL，后端处理逻辑几乎不需要改变，最后只需要将分页后的数据添加到原来页面的尾部即可。

### 动手实现
首先，我们需要将页面结构重新组织一下，大致变为下面这个样子：

```html
<table>
  <!-- 将包含数据的内容页面移入partial中 -->
  <tbody id="内容页面id">
    <%= render partial: "partial页面" %>
  </tbody>
</table>

<%= will_paginate @分页实例变量 %>
```
接下来按照思路实现一下鼠标滚轮滚动到页面底部的效果：

```javascript
$(window).scroll(function(){
    //检测鼠标滚轮是否已经滚动到页面底部
    if($(window).scrollTop() > ($(document).height() - $(window).height() - 50)){
        //AJAX请求will_paginate的下一页的href
        $.getScript($('.pagination .next_page').attr('href'));
    }
});
```
这样AJAX请求已经能够发送到后端了，剩下的工作是修改一下controller执行后对应的`.js.erb`中的代码：

```ruby
# 注意这里是append方法
$('#内容页面id').append('<%= j render(partial页面)%>');

# 需要更新一下底部的will_paginate页面的翻页按钮
$('.pagination').html('<%= j will_paginate(@分页实例变量) %>');
```
现在测试一下，会发现当我们鼠标滚轮滑动到页面底部时，确实会向后台发送AJAX请求了，但是却会连续发送多次重复的请求。现在需要回头检查一下javascript代码：

当我们通过`getScript`发送请求时，仍旧在滚动滚轮，这样`$(window).scroll`函数依然会被触发，当新的一页数据没有被加载出来时，if判断语句始终为true，所以会导致发送多次AJAX请求。

我们需要修改一下刚才的javascript代码：

```javascript
$(window).scroll(function(){
    //先取出下一页的url
    url = $('.pagination .next_page').attr('href');
    //将url作为if的一个判断条件，保证当前页面滚动到底部时只会触发一次ajax请求
    if(url && $(window).scrollTop() > ($(document).height() - $(window).height() - 50)){
        //这里将pagination的url替换成文字
        $('.pagination').text("加载中...");
        $.getScript(url);
    }
});
```

再测试一下，会发现已经能够正常滚动加载了，但是当我们滚动到最后一页时，会看到`will_paginate`的翻页组件露了出来，稍稍修改一下我们的`.js.erb`：

```ruby
<% if @分页实例变量.next_page %>
  $('.pagination').html('<%= j will_paginate(@分页实例变量) %>');
<% else %>
  $('.pagination').hide();
<% end %>
```

看起来差不多了。但是我们还需要考虑一种极端情况，如果页面中每页展示的数据很少，比如是下图这个样子，连滚动条都没有：

![没有滚动条页面示例]({{ site.baseurl }}/images/scroll_to_load.png "没有滚动条页面示例")

这种情况下`$(window).scroll`函数永远也不会被触发，所以我们还需要在前端javascript代码中补充一句：

```javascript
$(window).scroll();
```

这样，我们就实现了一个完整的下拉滚动异步加载效果。

### 参考
[Endless Page](https://www.youtube.com/watch?v=PQX2fgB6y10)
