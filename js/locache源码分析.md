# locache[源码](https://github.com/d0ugal-archive/locache/blob/0.4.0/locache.js)分析

> 以0.4.0版本为例。

```
// Object context bnding shim to support older versions of IE.
var bind = function (func, thisValue) {
    return function () {
        return func.apply(thisValue, arguments);
    };
};

```
该代码主要是绑定函数执行时，当前this的指向，和xx.bind(this)功能一样。其实就是es5 中新增的bind()方法的具体实现，保证浏览器兼容。

## 从整体的角度看locache的源码，主要是如下结构：
- a.定义构造方法；
```
function LocacheCache(options){
  //...
}
```
- b.为LocacheCache定义各种需要的原型属性和方法。

- c.最后声明locache变量，值为new LocacheCache()，将locache绑定到window上。正常情况下，该js在页面中只会执行一次，所以，window.locache 的值是单例的。

## 分析[locache](https://github.com/d0ugal-archive/locache/blob/0.4.0/locache.js)其中异步模块代码

```javascript
// A defer implementation to avoid IO access blocking the current
// thread. This is exposed on the LocacheCache prototype, simply so it
// can be accessed from within the unit tests. It's not intended for
// public use.
var defer = (LocacheCache.prototype._defer = (function() {
  // Store of the pending functions
  var timeouts = [];
  // Message name used to identify posts related to the defer
  var messageName = "function-defer-message";

  // Add a message event listener. If its not from this window or doesn't
  // have the message name defined above, don't do anything and allow it
  // to propagate to other handlers. Otherwise, its meant for us so stop
  // the event.
  var addEventListener;
  if (root.addEventListener) {
    addEventListener = root.addEventListener;
  } else if (root.attachEvent) {
    addEventListener = root.attachEvent;
  }

  addEventListener(
    "message",
    function(event) {
      if (event.source !== root || event.data !== messageName) {
        return;
      }
      event.stopPropagation();

      // Make sure we have some pending functions, otherwise return.
      if (timeouts.length === 0) {
        return;
      }

      // take the oldest from the 'queue' and call that functions.
      var fn = timeouts.shift();
      fn();

      // Return false, to make sure the event isn't propagated
      return false;
    },
    true
  );

  // Constructor for the Defer, takes a function object and stores it
  // on itself.
  function Deferred(fn) {
    this.fn = fn;
  }

  // The defer method runs the function and stores the result. Upon
  // finishing, it looks for a finishedFunction, that if exists, is
  // called and passed the result.
  Deferred.prototype.defer = function() {
    this.resultValue = this.fn();
    if (this.hasOwnProperty("finishedFunction")) {
      this.finishedFunction(this.resultValue);
    }
  };

  // Return if the defer has finished. THis is determined by the
  // existence of the resultValue on the object.
  Deferred.prototype.hasFinished = function() {
    return this.hasOwnProperty("resultValue");
  };

  // The finished function takes another function and assigns that to
  // be called when the deferred function has finished.
  Deferred.prototype.finished = function(fn) {
    this.finishedFunction = fn;
    // Check to see if the deferred function has finished, this can
    // happen if its very quick or the finished function is assigned
    // late. If it is, call it straight away.
    if (this.hasFinished()) {
      this.finishedFunction(this.resultValue);
    }
    // Make this object chainable.
    return this;
  };

  // Wrapper utility function that provides access to the Deferred
  // implementation and makes it very simple to use.
  function defer(fn) {
    // Create the defer instance that wraps the function
    var d = new Deferred(fn);
    // Add the defer method on the Deferred object, with the instance
    // bound to the queue.
    timeouts.push(bind(d.defer, d));
    // post a message to the window that can be received by the
    // message handler.
    root.postMessage(messageName, "*");
    // Finally, return the deferred object instance.
    return d;
  }

  // Only return the defer function, this is all we want to be publicly
  // accessible.
  return defer;
})());

```
- (...)。括号里面的代码直接执行。括号里面所以内容用逗号,分隔开。不能进行变量申明，可以进行赋值，计算，打印,函数执行等，并输出最后以后逗号中的结果。

```
var defer = (LocacheCache.prototype._defer = (function() {}
  //...
))
```

> 该段代码表示，将后面的立即执行函数的结果赋值给LocacheCache.prototype._defer。由于(...)这种写法，将赋值的结果又传给了defer。所以，相当于 defer = LocacheCache.prototype._defer = (function() {}))

- 判断当前浏览器环境是否支持某种api时，可以通过try catch方法进行判断，如源码[79行](https://github.com/d0ugal-archive/locache/blob/0.4.0/locache.js#L79)。