---
title: angular自定义filter
date: 2016-02-05 20:01:53
tags: [angular,angularjs,前端]
categories:
- 技术博客
- 原创
---
{: id="20210312121354-mtdya60"}

> angular除了几个自带的常用的filter,还可以自定义filter,以满足不同的需求，简单研究了一下自定义filter，记录一下。
> 有如下场景，后台返回的数据中，status可能是英文字符串，如果在html中使用if进行挨个判断，则显得有些啰嗦，这样我们就可以使用自定义的filter实现
> {: id="20210312121354-kd27aoe"}
{: id="20210312121354-4b251y8"}

<!-- more -->

javascript代码：
{: id="20210312121354-u0vvgdz"}

```javascript
var myapp = angular.module('demoApp', []);
myapp.controller('filterController',['$scope',function($scope){
    $scope._status= ['ENABLE','DISABLED','BINDING'];
}]);
myapp.filter('statusFilter',function(){
    return function(input){
        if(input === 'ENABLE') return '启用';
        if(input === 'DISABLED') return '停用';
        if(input === 'BINDING') return '绑定';
}
});
```
{: id="20210312121354-ufvjmfb"}

html代码：
{: id="20210312121354-z3fumwb"}

```html
<p ng-repeat="status in _status" >{{status | statusFilter}}</p>
```
{: id="20210312121354-orcfhjh"}

在controller中我们定义了一个数组_status,用来模拟后台返回的数据，我们在前台显示的时候可能不希望直接显示为英文的，客户看不懂啊。。。 如果在html中使用ng-if进行判断然后再显示，会显得有些啰嗦。我们直接写一个简单的filter，专门对status这个字段进行过滤即可。
{: id="20210312121354-unbe3um"}


{: id="20210312121354-je3of3l" type="doc"}
