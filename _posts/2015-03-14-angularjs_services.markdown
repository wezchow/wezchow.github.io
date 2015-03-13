---
layout:         post
title:          "AngularJS - Services(Part-I)"
date:           2015-03-14
categories:     AngularJS
tags:           AngularJS
image:          /assets/article_images/2014-11-30-mediator_features/night-track.JPG
---


<br/>
这几天解决了几个存储层面的问题之后，开始继续折腾前端。自己的玩具前端选择了使用 AngularJS, 逐步深入的过程中愈发感觉 Services, Factories 和 Providers 这几个概念有点搞，花点时间来弄弄清楚吧。以下是 Tyler Mcginnis 在其个人博客上面的阐述，直接拿来翻译一下加深印象 ([原文链接](http://tylermcginnis.com/angularjs-factory-vs-service-vs-provider/))。


## 前言
当我们刚开始接触 AngularJS 的时候，你不经意间可能会发现在你的 contollers 和 $scope 里面充斥着洪水般混乱且多余的逻辑。所以我们应当尽可能早的意识到 controller 应该保持一种非常瘦的状态 (thin)，绝大多数的业务逻辑和数据持久化应该交由 Service 来处理或储存 <span style="background-color: #FFE700;">(所幸老夫写第一个 controller 的时候就有类似的疑问... )</span>。某一天，我在 Stack Overflow 上反复看到有人在尝试使用 controller 来持久化数据，那绝对不是使用 controller 的初衷。让我们回忆一下经典 MVC 场景中的 Controller, 它应该只有在需要时被实例化，反之也就意味着它不应当继续存在。正因为如此，每当你的 AngularJS 程序切换路由 (route) 或是重载页面时，当前的 controller 总是会被清理干净。而 Services 则提供了一种在程序生命周期内可以保存数据，可以跨越多个 controllers 被重复使用的途径。


AngularJS 为我们提供了三种方法来创建和注册我们自己的 Service.

1. Factory
2. Service
3. Provider

### 简介
1. 当你使用 <strong>Factory</strong> 时，你会先创建一个对象，然后给它添加属性，最后返回这个对象本身。当你把这个 Service 传递给你的 controller, 那些你创建的 factory 属性会在第一时间生效。

{% highlight javascript %}
app.controller('myFactoryCtrl', function($scope, myFactory) {
  $scope.artist = myFactory.getArtist();
});

app.factory('myFactory', function() {
  var _artist = '';
  var service = {};

  service.getArtist = function() {
    return _artist;
  }

  retrun service;
});
{% endhighlight %}

2. 当你使用 <strong>Service</strong> 时，它自身会使用关键字 "new" 来实例化。因为如此，你需要针对 "this" 添加对象属性，并且 Service 也会返回 "this"。当你把这个 Service 传递给你的 controller，你所针对 "this" 添加的属性会在第一时间生效。

{% highlight javascript %}
app.controller('myServiceCtrl', function($scope, myService) {
  $scope.artist = myService.getArtist();
});

app.service('myService', function() {
  var _artist = 'Nelly';

  this.getArtist = function() {
    return _artist;
  }
});
{% endhighlight %}

3. <strong>Providers</strong> 是唯一一种可以传递给 .config() 的 service. 使用 provider 的一个场景是在你希望针对 service 对象还没有实际可用前，提供 module 范围的配置。

{% highlight javascript %}
app.controller('myProviderCtrl', function($scope, myProvider) {
  $scope.artist = myProvider.getArtist();
  $scope.date.thingFromConfig = myProvider.thingOnConfig;
});

app.provider('myProvider', function() {
  this._artist = '';
  this.thingFromConfig = '';

  // 只有从 $get 方法返回的属性才能在 controller 中使用
  this.$get = function() {
    var that = this;
    return {
      getArtist: function() {
        return that._artist;
      },
      thingOnConfig: that.thingFromConfig
    }
  }
});

app.config(function(myProviderProvider) {
  myProviderProvider.thingFromConfig = 'This was set in config()';
}
{% endhighlight %}

<br/><br/>

### 详细说明
为了可以更详细的说明 Factory, Service 和 Provider 的区别，我们将会分别使用这三者打造一个同样功能的服务。这个服务会使用到 iTunes API 以及使用 $q 的 promises。

#### Factory
Factories 可能是最流行的创建和配置 service 的方法。一个更简洁基本的范例如下：
{% highlight javascript %}
app.factory('myFactory', function() {
  var service = {};
  return service;
});
{% endhighlight %}

现在不管我们赋予这个 'service' 何种 properties 都会在传递给我们的 controller 时可用。让我们给我们的回调函数添加一些 'private' 变量。这些私有变量不会被 controller 直接访问到，但是我们可以设置一些 getter/setter 方法在我们需要的时候取到这些私有变量的值。

{% highlight javascript %}
app.factory('myFactory', function() {
  var service = {};
  var baseUrl = 'https://itunes.apple.com/search?term=';
  var _artist = '';
  var _finalUrl = '';

  var makeUrl = function() {
    _artist = _artist.split(' ').join('+');
    _finalUrl = baseUrl + _artist + '&callback=JSON_CALLBACK';
    return _finalUrl;
  }

  return service;
});
{% endhighlight %}

这里你可能发现我们并没有把那些变量直接和 'service' 联系起来。我们只是简单的创建了他们，为的只是方便之后的使用和修改。

- <strong>baseUrl</strong> 是 iTunes API 需要的最基本的链接
- <strong>_artist</strong> 是我们希望查找的艺人
- <strong>_finalUrl</strong> 是我们最终得到的链接用来发起 API 调用
- <strong>makeUrl</strong> 函数用来创建并返回我们的 iTunes URL

现在我们的 helper/private 变量和相关函数已经准备就绪，让我们动手给 'service' 对象增加一些属性吧。我们会创建 setArtist 和 getArtist 方法用来简单的设置或返回艺人信息。同时我们也会添加一个方法使用我们自己生成的 URL 调用 iTunes API。这个方法将会在数据完整被远程返回后，返回一个 promise。假如你对 promise 并不非常熟悉，我非常建议你可以深入学习一下它。

- <strong>setArtist</strong> 会保存艺人信息并允许你修改它
- <strong>getArtist</strong> 返回艺人信息
- <strong>callItunes</strong> 首先会调用 makeUrl() 方法构建 $http request 所需要的 URL. 接着它会装载一个 promise 对象，利用 $http request 调用我们最终生成的 URL, 因为 $http 返回的也是一个 promise, 所以连接在这个调用之后的 .success 或 .error 方法可以被执行。最后我们会解析这个返回的 promise 检查是否返回正常，如有错误则发送一条 "There was an error" 的信息。

{% highlight javascript %}
app.factory('myFactory', function($http, $q) {
  var service = {};
  var baseUrl = 'https://itunes.apple.com/search?term=';
  var _artist = '';
  var _finalUrl = '';

  var makeUrl = function() {
    _artist = _artist.split(' ').join('+');
    _finalUrl = baseUrl + _artist + '&callback=JSON_CALLBACK';
    return _finalUrl;
  }

  service.setArtist = function(artist) {
    _artist = artist;
  }

  service.getArtist = function() {
    return _artist;
  }

  service.callItunes = function() {
    makeUrl();
    var deferred = $q.defer();

    $http({
      method: 'JSONP,
      url: _finalUrl
    }).success(function(data) {
      deferred.resolve(data);
    }).error(function() {
      deferred.reject('There was an error');
    })
    return deferred.promise;
  }

  return service;
});
{% endhighlight %}

到这里我们的 factory 就完成了，现在我们可以把 myFactory 注入到任何 controller 中，并且 service 对象中的 setter/getter/callItunes 都可以被使用的 controller 调用的到。

{% highlight javascript %}
app.controller('myFactoryCtrl', function() {
  $scope.data = {};
  $scope.updateArtist = function() {
    myFactory.setArtist($scope.data.artist);
  }

  $scope.submitArtist = function() {
    myFactory.callItunes()
      .then(function(data) {
        $scope.data.artistData = data;
      }, function(data) {
        alert(data);
      })
  }
});

在上面的 controller 中，我们注入了 'myFactory' 服务。接着利用 'myFactory' 所返回的数据来赋值给 $scope 对应的对象。如果你从来没有接触过 promises，这里会有一个小把戏。因为 callItunes 会返回一个 promise, 所以我们可以使用 .then() 方法在 promise 是一个完整合法数据的时候，赋值给 $scope.data.artistData。你会发现在这个例子中我们的 controller 非常瘦。所有逻辑和数据持久都放在了我们的 service 中。

Too long, To be continue ...

