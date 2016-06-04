---
layout: post
title: Angular `ng-strict-di`
date: '2016-06-03T23:47:00.001-05:00'
author: Phillip Green II
tags:
- angular
modified_time: '2016-06-03T23:47:00.001-05:00'
---

The dependency injection in AngularJS is amazing.  It allows code to be broken down into very isolated parts that improve both maintenance and testability.  In this post, I'm discussing a common pitfall that happens when angular `1.x` is [minified][wiki-minification].

To illustrate the problem, we will start with a simple application:

```javascript
angular.module('plunker', [])
  .service('myService', function($http) {
    return {
      lookupYoda: function() {
        return $http({
            method: 'GET',
            url: '//swapi.co/api/people/20/'
          })
          .then(function successCallback(response) {
            return response.data;
          });
      }
    };
  })
  .controller('MainCtrl', function(myService, $scope) {
    $scope.name = 'World';
    myService.lookupYoda().then(function(yoda) {
      $scope.name = yoda.name;
    });
  });
```

You can see a working version of this code on [here][plunker-not-minified-not-strict].

The code works fine and there is nothing obviously incorrect about it.  The problem stems from how the dependencies are specified.  The code above defines the dependencies implicitly.  At runtime, angular will use reflection to determine the names of the parameters that are injected.  

As development teams become more adept and applications grow larger, there comes a point where applications are minified to improve performance.  When this happens, the name of the parameters will no longer match the desired dependencies.  The following example shows how the previous code could have been minified.

```javascript
!function(n){n.module("plunker",[]).service("myService",function(n){return{
lookupYoda:function(){return n({method:"GET",url:"//swapi.co/api/people/20/"})
.then(function(n){return n.data})}}}).controller("MainCtrl",function(n,o){
o.name="World",n.lookupYoda().then(function(n){o.name=n.name})})}(angular);
```
You can see a "working" version of this code at [here][plunker-minified-not-strict].

When you run the plunker with the minified code, you will notice that the preview doesn't render correctly and if you check the console, you will find the following error:

```
angular.js:12722 Error: [$injector:unpr] Unknown provider: nProvider <- n <- MainCtrl
http://errors.angularjs.org/1.4.9/$injector/unpr?p0=nProvider%20%3C-%20n%20%3C-%20MainCtrl
    at angular.js:68
    at angular.js:4346
    at Object.getService [as get] (angular.js:4494)
    at angular.js:4351
    at getService (angular.js:4494)
    at Object.invoke (angular.js:4526)
    at extend.instance (angular.js:9380)
    at nodeLinkFn (angular.js:8497)
    at compositeLinkFn (angular.js:7929)
    at compositeLinkFn (angular.js:7932)
```

This happens because `$http` was renamed to `n` and there is no dependency called `n`.  In this small example it is easy to locate the problem.  As applications grow larger and larger, locating the problem becomes more and more difficult.  

To make this particular error more complicated, it may not occur in the developer's environment.  It is a common practice to not minify code during development.  This keeps the redeploy cycle very quick.  Minification will only happen on the continuous integration server.  This leads to the situation where everything works fine in the developer's environment but not in the testing or QA environments.

To prevent this error from happening, explicit dependency injection should always be used.  To learn how, see either [Inline Array Annotation][angularjs-explicit-di-array] or [$inject Property Annotation][angularjs-explicit-di-annotation].

To help developers avoid using implicit dependency injection, angular 1.3 added [Strict DI Mode][angularjs-strict-dependency-injection].  This is enabled by adding `ng-stict-di` to the same element as `ng-app`:

```html
<html ng-app="plunker" ng-strict-di>
```

When this is added to the original non-minfied sample ([here][plunker-not-minified-strict]), the following error is generated:

```
angular.js:12722 Error: [$injector:strictdi] MainCtrl is not using explicit annotation and cannot be invoked in strict mode
http://errors.angularjs.org/1.4.9/$injector/strictdi?p0=MainCtrl
    at angular.js:68
    at Function.annotate [as $$annotate] (angular.js:3804)
    at Object.invoke (angular.js:4513)
    at extend.instance (angular.js:9380)
    at nodeLinkFn (angular.js:8497)
    at compositeLinkFn (angular.js:7929)
    at compositeLinkFn (angular.js:7932)
    at publicLinkFn (angular.js:7809)
    at angular.js:1682
    at Scope.$eval (angular.js:16251)
```

This error will happen in the developer's environment, so there are no minification surprises later.


## References
* [AngularJS: Strict DI Mode][angularjs-strict-dependency-injection]
* [Wikipedia: Minification (programming)][wiki-minification]


[angularjs]: <https://angularjs.org/> "AngularJS"

[angularjs-dependency-injection]: <https://docs.angularjs.org/guide/di> "AngularJS: Dependency Injection"

[angularjs-explicit-di-array]: <https://docs.angularjs.org/guide/di#inline-array-annotation> "AngularJS: Inline Array Annotation"

[angularjs-explicit-di-annotation]: <https://docs.angularjs.org/guide/di#-inject-property-annotation> "AngularJS: $inject Property Annotation"

[angularjs-strict-dependency-injection]: <https://docs.angularjs.org/guide/production#strict-di-mode> "AngularJS: Strict DI Mode"

[wiki-minification]: <https://en.wikipedia.org/wiki/Minification_(programming)> "Wiki: Minification (programming)"

[plunker-not-minified-not-strict]: <https://plnkr.co/edit/CUildt?p=preview> "Plunker: Not Minified, Not Strict DI"

[plunker-minified-not-strict]: <https://plnkr.co/edit/9I7gdJ?p=preview> "Plunker: Minified, Not Strict DI"

[plunker-not-minified-strict]: <https://plnkr.co/edit/IeS4Sv?p=preview> "Plunker: Not Minified, Strict DI"
