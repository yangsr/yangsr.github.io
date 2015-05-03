---
layout: post
title: 微信开放平台Omniauth探索
---

### 准备
最近在折腾网站的微信登录功能，最终希望实现的效果是类似[一号店这样的微信登录](https://passport.yhd.com/passport/login_input.do)。

为此首先需要在[微信开放平台](https://open.weixin.qq.com/)上注册开发者账号，创建网站应用，获得对应的AppID和AppSecret，并申请微信登录接口。在这个过程中，会被要求填写网站信息，包括授权回调域和官网网址。

这里需要注意的是授权回调域的填写，刚开始我是这么填写的：
{% highlight html %}
example.com/auth/wechat/callback
{% endhighlight %}
然而这么填写将会在后面的OAuth认证时提示redirect_uri参数错误。微信开放平台并没有给出授权回调域的官方填写说明，我是在[微信公众平台](https://mp.weixin.qq.com/)的开发者文档里看到了这么一段话：
> 关于网页授权回调域名的说明

> 请注意，这里填写的是域名（是一个字符串），而不是URL，因此请勿加http://等协议头；

> 授权回调域名配置规范为全域名，比如需要网页授权的域名为：www.qq.com，配置以后此域名下面的页面http://www.qq.com/music.html 、 http://www.qq.com/login.html 都可以进行OAuth2.0鉴权。

哦，原来改成这样就可以了：
{% highlight html %}
example.com
{% endhighlight %}

### 调试
接下来，我想的是到底该如何调试，因为授权回调域不能填成localhost，总不至于在线上调试吧。

搜索后发现有人推荐一款叫ngrok的软件，只需要注册并下载ngrok，就可以通过ngrok获得一个外网域名，而这个外网域名实际访问的是本地主机。

具体ngrok的注册下载流程省略，按照[ngrok官网](https://ngrok.com/)一步步来就可以了，这里需要自带梯子。

之后本地执行./ngrok http -subdomain=example 3000，出现下面的画面就算成功啦：
![ngrok]({{ site.baseurl }}/images/ngrok.png "ngrok")
那么开发阶段，就把刚才提到的授权回调域改成自己设定的ngrok地址吧。

### OAuth流程
整个微信登录的流程是如下图所示的：
![微信登录时序图]({{ site.baseurl }}/images/oauth_process.png "微信登录时序图")

OAuth过程可以使用omniauth，omniauth 是一个利用 Rack 中间件实现的灵活的认证系统。

omniauth需要结合具体平台的strategy使用，比如你需要做github账号的登录，那就需要[omniauth-github](https://github.com/intridea/omniauth-github)这个gem。omniauth官方列出了社区维护的各个平台的[strategy list](https://github.com/intridea/omniauth/wiki/List-of-Strategies)，我在list中找到了微信的omniauth strategy：[omniauth-wechat-oauth2](https://github.com/skinnyworm/omniauth-wechat-oauth2)。

具体怎么使用应该首先看看README的文档，我读到了这么一句话：
> You need to get a wechat API key at: http://mp.weixin.qq.com

这不是微信公众平台的地址么，可我需要的是微信开放平台的OAuth认证啊，这两个微信平台的OAuth流程是否一致，需要确定下。于是我开始打开了微信开放平台和微信公众平台的OAuth流程的文档进行对比，发现果然有些细节处不一致，比如第一步请求code时，两者请求地址的形式略有不同，微信开放平台的请求code地址是：
{% highlight html %}
https://open.weixin.qq.com/connect/qrconnect?appid=APPID&redirect_uri=REDIRECT_URI&response_type=code&scope=SCOPE&state=STATE#wechat_redirect
{% endhighlight %}
而微信公众平台的请求code地址却是：
{% highlight html %}
https://open.weixin.qq.com/connect/oauth2/authorize?appid=APPID&redirect_uri=REDIRECT_URI&response_type=code&scope=SCOPE&state=STATE#wechat_redirect
{% endhighlight %}

我去查看了omniauth-wechat-oauth2的strategy源码：
{% highlight ruby %}
# /lib/omniauth/strategies/wechat.rb
option :client_options, {
  site:          "https://api.weixin.qq.com",
  authorize_url: "https://open.weixin.qq.com/connect/oauth2/authorize#wechat_redirect",
  token_url:     "/sns/oauth2/access_token",
  token_method:  :get
}
{% endhighlight %}
这个authorize_url的地址显然是微信公众平台请求code时的格式，这就意味着这个gem并不是我需要的。

不过我发现微信公众平台和微信开放平台的基本OAuth流程是类似的，于是我参考omniauth-wechat-oauth2的源码，自己实现了微信开放平台的omniauth strategy，并托管在了[GitHub上](https://github.com/yangsr/omniauth-wechat-oauth2)。这里感谢一下omniauth-wechat-oauth2的原作者skinnyworm，复用了大部分他的代码。

### CSRF错误
剩下的工作就和其他平台的OAuth流程差不多了，这里就不再赘述，具体可以看一下文章后的参考链接。

只是这里遇到了一个CSRF错误，特别提一下：
{% highlight ruby %}
Started GET "/auth/wechat/callback?code=xxxxxxxx&state=xxxxxxxx" for 183.157.160.37 at 2015-04-28 22:26:48 +0800
I, [2015-04-28T22:26:48.017297 #18928]  INFO -- omniauth: (wechat) Callback phase initiated.
E, [2015-04-28T22:26:48.017785 #18928] ERROR -- omniauth: (wechat) Authentication failure! csrf_detected: OmniAuth::Strategies::OAuth2::CallbackError, csrf_detected | CSRF detected
E, [2015-04-28T22:26:48.017891 #18928] ERROR -- omniauth: (wechat) Authentication failure! invalid_credentials: OmniAuth::Strategies::OAuth2::CallbackError, csrf_detected | CSRF detected
{% endhighlight %}
这是因为我本地调试时习惯性地打开了localhost:3000，而我又开启了ngrok，调试时应该直接打开example.ngrok.io的。

### 参考
railscasts-china的[Omniauth 1](http://railscasts-china.com/episodes/omniauth-1)

happypeter的[login-with-linkedin](http://haoduoshipin.com/episodes/98)

感兴趣的同学还可以拓展阅读下[微信开放平台和微信公众平台的区别](http://www.zhihu.com/question/21074751)
