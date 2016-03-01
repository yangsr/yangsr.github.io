---
layout: post
title: 聊聊HTML5的histroy相关API
---

### 平时没有注意到的浏览器URL细节
Ajax技术能够大大提升网站的用户体验，我们的平台中也包括了诸多Ajax请求。这次遇到的需求场景是：当用户点击一个Tab标签，发送一次Ajax请求，服务器端给出响应后，浏览器的URL并不会对应地做出改变。而用户恰恰希望能够获取到最新的URL，这样就可以把这个URL发送给别人，别人点击该URL后进入平台看到对应的界面。

类似的体验你一定在Github上遇到过：当我们在Github上浏览一个项目下的文件时，一层层文件夹点击进去，虽然发送的都是Ajax请求，但是如果细心观察一下，浏览器的URL也是在相应改变的。

![Github示意]({{ site.baseurl }}/images/github-example.png "Github示意图")

### history.pushState方法
要实现这个效果，其实只需要在你的javascript click事件代码中添加一句话就可以了：

```javascript
  history.pushState(null, "", this.href);
```

我们来看看文档上是怎么描述这个方法的作用的吧：

> 这将让浏览器的地址栏显示href，但不会加载href页面也不会检查href是否存在。

pushState方法接收三个参数：

* 第一个参数叫状态对象(state object)，任何可序列化的对象都可以，最大640K。当popstate事件被触发时，事件对象的state属性包含历史记录条目的状态对象的拷贝。

* 第二个参数当前会被浏览器忽略，所以可以暂时不去关心它。

* 第三个参数就是对应的URL，既可以是绝对路径，也可以是相对路径，但必须与当前URL同源，否则将抛出异常。

### popstate事件
到目前为止，我们已经解决了发送Ajax请求URL也会相应改变的问题。我们再试试点击浏览器的回退按钮，页面并没有正常地回退，对吧？

看看我们刚才提到的popstate事件，到它发挥作用的时候啦：

```javascript
$(window).bind("popstate", function () {
  // 需要注意的是，getScript发送的是AJAX请求
  $.getScript(location.href);
});

```
每当激活的histroy entry发生变化时，都会触发popstate事件。当我们点击后退按钮的时候，浏览器的URL其实已经改变了，只是页面没有变化，于是我们只要执行getScript方法，并且把当前浏览器的URL传递给它就好啦。

添加完上述代码后再试试，是不是能够正常后退了呢。

### 余下的工作
如果你和我一样使用Rails作为Web开发框架，那么你现在会发现还需要处理额外的工作：即在controller层相应的action中添加对HTML请求的处理。

当我们通过Ajax请求进入的新页面后，按F5刷新，请求的将是更新后的URL，这时发送的就不再是Ajax请求，而是一个正常的HTML请求，在controller层找不到对应的请求处理方式的话，将会报错。

### 参考
[Rails Cast](http://railscasts.com/episodes/246-ajax-history-state?view=asciicast)

[mozilla History_API 官方文档](https://developer.mozilla.org/en-US/docs/Web/API/History_API)