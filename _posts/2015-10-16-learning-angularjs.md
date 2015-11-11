---
layout: post
title: AngularJs的那些坑
category : javascript
tagline: "learning angularjs"
tags : [javascript, AngularJs]
---
{% include JB/setup %}

### 用AngularJS开发过程中遇到了无数的坑，一路磕磕绊绊走来，满满都是泪...
# 当然，也乐在其中~
![manong](https://cloud.githubusercontent.com/assets/3291617/10533537/3a1228a6-73fd-11e5-9f7e-48965d995da0.gif)

在这里把这些遇到的坑都记录下来，便于日后查找，也给大家提个醒，持续更新中

## 1. AngularJS 中$http post和jQuery中的ajax post的区别，以及和PHP的结合
AngularJS 中的$http post方法和jQuery中的ajax post方法使用的头信息不同，ng中Content-Type 是application/json，jq中是application/x-www-form-urlencoded，对于PHP来说，jq中的处理比较方便，直接用$_POST就能获取，但是ng默认发送的是json数据，PHP不能直接获取。
解决办法无非两种：从前端（AngularJS）解决和从后端（PHP）解决

### (1). 从前端解决:
在ng模块或者config()方法中修改post头信息：
{% highlight javascript %} 
$httpProvider.defaults.headers.post['Content-Type'] = 'application/x-www-form-urlencoded;charset=utf-8';
$httpProvider.defaults.headers.put['Content-Type'] = 'application/x-www-form-urlencoded;charset=utf-8';
{% endhighlight %}

单单修改头信息还不够，还要增加一个拦截器，用于把json数据序列化成 a=123&b=456的格式：
{% highlight javascript %}
$httpProvider.interceptors.push(['$q', function($q) {
  return {
    request: function(config) {
      if (config.data && typeof config.data === 'object') {
        config.data = serialize(config.data);
      }
        return config || $q.when(config);
    }
  };
}]);
{% endhighlight %}
序列化函数（注意如果参数是对象需要递归处理）：
{% highlight javascript %}
var serialize = function(obj, prefix) {
    var str = [];
    for (var p in obj) {
        var k = prefix ? prefix + "[" + p + "]" : p, 
              v = obj[p];
        str.push(typeof v == "object" ? serialize(v, k) : encodeURIComponent(k) + "=" + encodeURIComponent(v));
      }
    return str.join("&");
}
{% endhighlight %}
### (2). 从后端解决（我比较喜欢第二种方法，因为json格式的数据对前端来说很方便）
需要在PHP代码开头加入下面代码：
{% highlight php %}
<?php 
$content_type_args = explode(';', $_SERVER['CONTENT_TYPE']);
if ($content_type_args[0] == 'application/json') {
    $_POST = json_decode(file_get_contents('php://input'), true);
    $_REQUEST = array_merge($_REQUEST, $_POST);
}
?>
{% endhighlight %}

## 2. 在模板中方便的从1到n循环
ng模板中的对象、数组循环功能很强大，但最简单的从1-n的数字循环却实现不了，所以需要自己写一个过滤器
{% highlight javascript %}
angular.module('myApp.filters', [])
    .filter('range', function() {
        return function(input, start, total) {
            start = angular.isNumber(start) ? parseInt(start) : 0;
            total = angular.isNumber(total) ? parseInt(total) : 0;
            for (var i = start; i <= total; i++)
                input.push(i);
            return input;
        };
    });
{% endhighlight %}

用的时候只需这样：
{% highlight html %}
      <select ng-options="i as i for i in []|range : 0 : 17"></select>
{% endhighlight %}
## 3. 模板中数据绑定输出HTML代码
出于安全考虑，ng-bind 是会把绑定的数据进行HTML转义的，如果想直接输出HTML需要写一个过滤器：
{% highlight javascript %}
angular.module('myApp.filters', [])
    .filter('toTrusted', ['$sce', function ($sce) {
        return function (text) {
            return $sce.trustAsHtml(text);
        }
    }])
{% endhighlight %}
使用过滤器：
{% highlight html %}
<span ng-bind-html="desc|toTrusted"></span>
{% endhighlight %}

## 4. 路由切换时自动跳到页面顶部
假如路由改变、视图刷新时页面不是在顶部位置，则新的视图页面也不在顶部位置，大多数时候我们希望它能自动跳到顶部，可以在模块run()方法中加入下面代码：
{% highlight javascript %}
angular.module('myApp')
    .run(['$rootScope', '$document', function($rootScope, $document) {
            $rootScope.$on('$routeChangeSuccess', function() {
                $document[0].body.scrollTop = $document[0].documentElement.scrollTop = 0;
            });
        }]);
{% endhighlight %}

## 5. AngularJS使用CORS发送跨域请求
用CORS发送跨域请求比JSONP更安全更简洁
在config中加入下面两行：
{% highlight javascript %}
angular.module('myApp')
    .config([ '$httpProvider', function($httpProvider) {
        $httpProvider.defaults.useXDomain = true;
        delete $httpProvider.defaults.headers.common['X-Requested-With'];
        }]);
{% endhighlight %}
同时需要服务器配合，PHP中加入：
{% highlight php %}
<?php 
//允许所有域名都请求
header("Access-Control-Allow-Origin:*");
//上线后为了安全起见应指定域名
//header("Access-Control-Allow-Origin:http://example.com");
?>
{% endhighlight %}

## 6. 控制按钮响应频率
考虑到有这样一个使用场景：
> 
![room-num-btn](https://cloud.githubusercontent.com/assets/3291617/10536724/a16c09fa-741f-11e5-8d78-e751f7b59c80.png)

页面上有两个按钮，点加号增加一个房间，点减号减少一个房间，房间数每发生一次变化都要重新发送一个ajax请求获取最新的房间价格，这个请求是比较耗时而且消耗服务器资源的，如果用户想选择3间房，他需要点击两次“+”号，如果第1次点加号时也发送一次请求是毫无意义的，白白增加服务器压力，而且异步请求也不能保证第2次点加号时的返回一定比第1次点加号时的返回晚，所以还有可能造成价格显示错误，解决办法是设置一个时间戳记录用户上一次请求时间，如果上一次请求时间小于n秒，则不响应用户事件，避免无意义的多次请求。

{% highlight javascript %}
angular.module('myApp.controllers')
    .controller('CreateOrderController', ['$scope', '$timeout',
        function($scope, $timeout) {
            var maxRoomNum = 8,
                roomChangeTimestamp = null,  //记录上次点击的时间戳
                updateResponseDelay = 1000,
                defRoomInfo = {
                    customer : {
                        firstName : '',
                        lastName : ''
                    },
                    adultNum : 1,
                    childrenInfo : []
                };
            //存储房间信息
            $scope.roomInfos = [];
            //增加房间
            $scope.reduceRoom = function() {
                if ($scope.roomInfos.length > 1) {
                    $scope.roomInfos.pop();
                    roomInfoChange(updateResponseDelay);
                }
            };
            //减少房间
            $scope.addRoom = function() {
                if ($scope.roomInfos.length < maxRoomNum) {
                    var roomInfo = angular.copy(defRoomInfo);
                    $scope.roomInfos.push(roomInfo);
                    roomInfoChange(updateResponseDelay);
                }
            };

            function roomInfoChange(updateResponseDelay) {
                roomChangeTimestamp = new Date().getTime();
                var currTimestamp = roomChangeTimestamp;
                $scope.$broadcast('roomInfo:willChange');
                $timeout(function() {
                    //上次点击之后没有发生新的点击，则广播事件
                    if (currTimestamp == roomChangeTimestamp) {
                        $scope.$broadcast('roomInfo:change');
                    }
                }, updateResponseDelay);
            }

            //入住信息发生变化，需要重新请求接口获取价格
            $scope.$on('roomInfo:change', function(e) {
                getNewPrice();
            });

            //发送ajax请求获取最新价格
            function getNewPrice() {
                   //代码略...
            }
}]);
{% endhighlight %}
模板代码：
{% highlight html %}
<a ng-click="reduceRoom()"></a>
<strong>{{roomInfos.length}}</strong>
<a ng-click="addRoom()"></a>
{% endhighlight %}