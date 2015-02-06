---
title: Using AngularJS, Google Chrome extensions, and the ShadowDOM
layout: post
---

# Using AngularJS, Google Chrome extensions, and the Shadow DOM

I have been working on a Google Chrome extension that will display either
the web page that the user is viewing, or replace it with something else, given
a certain condition. Rather than muck around with overlays, or wrapping the page
content in a div to be hidden and revealed, I elected for something much
simpler, and seemingly suited perfectly to the job at hand: the [Shadow
DOM](http://www.w3.org/TR/shadow-dom/).

## Shadow DOM

The Shadow DOM specification is (as of February 2015) a work in process, but has
been implemented in some browsers, including Google Chrome.

The Shadow DOM allows you to take a host element, say a particular div, the body,
or even the document element itself, and create a shadow root.

### Creating a shadow root

{% highlight javascript %}
var shadowRoot = document.documentElement.createShadowRoot();
{% endhighlight %}

Executing the expression above will cause the (compliant) browser to immediately
render the contents of the newly created shadow root. Since this newly
created shadow root is empty, that is exactly what the browser renders.

### Projecting content from the host

You can select content to project from the host element on to the shadow root
with the `content` tag and `select` attribute.

{% highlight html %}
<content select='*'></content>
{% endhighlight %}

Injecting the fragment above in to the shadow root will cause the browser to
project the entire contents of the host (the `documentElement`) on to the
shadow root. It is not difficult to imagine how we might use this to
switch back and forth between rendering the original web page and
something else.

## Using Angular to switch between original and something else

No need to rely on our imaginations, however.

Let us say we are browsing on a website with the following HTML:

{% highlight html %}
<!DOCTYPE html>
<html>
<body>
    <p>Hello, world!</p>
</body>
</html>
{% endhighlight %}

Our Google Chrome extension has the following content script:

{% highlight javascript %}
var container, module, shadowRoot, viewURL;

container = document.createElement('span');
module = angular.module('myApp', []);
shadowRoot = document.documentElement.createShadowRoot();
shadowRoot.appendChild(container);
viewURL = chrome.runtime.getURL('view.html');

module.controller('MyCtrl', ['$scope', function ($scope) {
    $scope.state = 'off';
    $scope.$on('switch', function () {
        $scope.state = $scope.state === 'off' ? 'on' : 'off';
    });
}).run(['$rootElement', '$rootScope', function ($rootElement, $rootScope) {
    $rootElement.html('<ng-include src="\'' + viewURL + ''\'" />');
    chrome.runtime.onMessage(message, sender, sendResponse) {
        if ('switch' === message) {
            $rootScope.$broadcast(message);
            $rootScope.$apply();
        }
    });
}]);

angular.bootstrap(container, [module]);
{% endhighlight %}

The Google Chrome extension has the following view for the shadow root:

{% highlight html %}
<span ng-controller="MyCtrl">
    <span ng-switch="switch">
        <span ng-switch-when="on">
            <p>Goodbye, world!</p>
        </span>
        <span ng-switch-when="off">
            <content select="*"></content>
        </span>
    </span>
</span>
{% endhighlight %}

## Source

[angularjs-chromeextension-shadowdom-example](https://github.com/maxenglander/angularjs-chromeextension-shadowdom-example)
