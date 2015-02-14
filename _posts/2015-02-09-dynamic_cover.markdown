---
layout: post
title:  "封面 - 多彩一些"
date:   2015-02-09 14:34:25
categories: Jekyll Python SAE
tags:
image: /assets/article_images/2014-11-30-mediator_features/night-track.JPG
---

<br/>
在配置这套 Jekyll 模板的过程中，始终对每篇博文需要指定封面图片感到有点点不知足。为什么我不能直接利用一些线上美图为己用呢？既然想到就开工吧。


首先是简单解决方案的
TODO LIST：

## 1. 数据源
又微又软的搜索引擎好用不好用不在这里讨论，老夫就是看上你们家背景图了！嘿嘿... 哟西，要想个办法借(tou)过来。有事儿就得找谷大爷啊，果然 <a href="http://stackoverflow.com/questions/10639914/is-there-a-way-to-get-bings-photo-of-the-day" target="_blank">Stackoverflow</a> 上跟老夫一般对人家背景图流口水的也早已有之，神马XML, JSON, RSS，要啥有啥，下面贴上我拿到的 JSON 结构，省略了一些噪音，只看主要部分。
{% highlight json %}
{
    "images": [{
        "startdate": "20150208",
        "fullstartdate": "201502081600",
        "enddate": "20150209",
        "url": "http://s.cn.bing.net/az/hprichbg/rb/BlackButte_ZH-CN7659716324_1920x1080.jpg",
        ......
    }],
    "tooltips": {
        ......
    }
}
{% endhighlight %}

## 2. 包装数据
搞定了数据源，如何在我自己的页面载入呢？由于微软提供的 URL 返回的是 pure JSON string, 想要在自己的页面引用到这个数据还需要小小的改动，没办法，还是要走 backend 对数据进行包装。自己手头没有可用的服务器怎么办？没关系，app engine 哪家强？一家家扫过来。

- [Google App Engine](https://appengine.google.com/), 很好，很“墙”大，托 GFW 的福，今生不知是否还有缘一品其中滋味，老夫望眼欲穿然不得近身，在墙外好好待着吧。
- [百度开放云](http://bce.baidu.com/)，尼玛还需要邀请码，我这种小喽喽还是用不起这种高大上的货色，负分，滚粗！
- [Sina App Engine (SAE)](http://sae.sina.com.cn/), 虽说技术能力上没有上面两家强，好在亲民啊！语言支持 PHP, Python, Java, 对于我现在要解决的问题，这些都是无关紧要的，随便挑一个分分钟搞定。一直对 Python 颇有好感，外加这门语言在大数据处理领域的统御能力，老夫迟早要动手一试，这次也就劳驾您了吧。

选定平台，选定后台语言，就需要找一个 Python Restful API 的框架喽，临时抱佛脚看了下 SAE 所支持的第三方模块，和主流的 Python 框架，发现 Flask 非常合适我的需求。下面无耻滴引用一下官网上的介绍：

> Flask is a microframework for Python based on Werkzeug, Jinja 2 and good intentions. And before you ask: It's BSD licensed!

Bingo, 关键词是 Microframework, 但对于我的需求来讲已经是绰绰有余喽。SAE 预装了 Flask 框架，所以直接耍起来。代码很糙，实现很简单，不过咩达到目的就行，要做的事情只有一件，就是把原始数据转化为 JavaScript 对象，之后便于在自己的站点解析：

<strong>bingBgJSON.py</strong>
{% highlight python linenos %}
{
    __author__ = 'wesley'

    from flask import Flask
    import urllib2

    app = Flask(__name__)

    ## Get json string from target URL
    response = urllib2.urlopen('http://www.bing.com/HPImageArchive.aspx?format=js&idx=0&n=1&mkt=en-US')
    jsonString = response.read()

    ## Create new route for request and return wrapped javascript object
    @app.route('/bingbg.json', methods=['GET'])
    def get_JSON():
        return "bingBgJSON = " + jsonString

    if __name__ == '__main__':
        app.run(debug=True)
}
{% endhighlight %}

<strong>index.wsgi</strong>
{% highlight python linenos %}
{
    import sae
    from bingBgJSON import app

    application = sae.create_wsgi_app(app)
}
{% endhighlight %}


我说过分分钟搞定的吧，index.wsgi 文件是 SAE 的入口文件。需要利用他对我的实现实例化。好喽，通过 SVN 把代码上传到 SAE，可以测试了，等等，我先吐个槽，尼玛新浪能用高级点的 VCS 咩，弄个 Git 会死啊！清下嗓子回到正题，点下连接看结果：[http://wsandbox.sinaapp.com/bingbg.js](http://wsandbox.sinaapp.com/bingbg.js)

Job done!

## 3. 前端修改
这边的修改就简单了，直接把原来显示图片背景的 CSS 注释掉，给 <div> 标签加上 id="bingBG"。然后在 JavaScript 引用区加上：
{% highlight javascript%}
<script type="text/javascript" src="http://wsandbox.sinaapp.com/bingbg.js"></script>
<script>
    $(document).ready(function() {
            bingBgURL = bingBgJSON.images[0].url;
            document.getElementById('bingBG').style.backgroundImage = "url('" + bingBgURL + "')";
    });
</script>
{% endhighlight %}

好喽，重新刷一下页面，现在老夫的封面图片就跟 Bing.com 同步喽。至此，老夫的小站又往前走了那么一小步，比磋进步了那么一点点，其实还是磋。其实对于 Cover Image 还有些想法，比如可以增加点数据源，可以更智能的根据我输入的博文主题去搜索在线高质量图片资源自动匹配等等，hmmm... 等老夫有空继续深入哈...

>老夫语录:
>装，是一种美德！我们需要的就是不停装，装不停！（近日重用 Ubuntu 有感）

<img style="display: block; margin: 0 auto;" src="/assets/emoji/line/2/426@2x.png"/>