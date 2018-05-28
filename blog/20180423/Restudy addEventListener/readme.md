# Restudy addEventListener

我们知道`addEventListener`是用于将指定的监听器注册到EventTarget上，当该对象触发指定的事件时，指定的回调函数就会被执行。事件目标可以是一个文档上的元素 Document 本身，或者任何其他支持事件的对象 (比如 XMLHttpRequest)。

## 语法

``` js
target.addEventListener(type, listener, options);

target.addEventListener(type, listener, { capture: Boolean, bubbling: Boolean, once: Boolean});

target.addEventListener(type, listener, useCapture);
target.addEventListener(type, listener, [, useCapture, wantsUntrusted]); // Gecko/Mozilla only
```

### type

> 表示监听事件类型的字符串

这个大家都应该知道。

### listener

> 当所监听的事件类型触发时，会接收到一个事件通知（实现了 Event 接口的对象）对象。listener 必须是一个实现了 EventListener 接口的对象，或者是一个函数。

一般来说我们经常用的就是一个函数。

### useCapture

> Boolean，是指在DOM树中，注册了该listener的元素，是否会先于它下方的任何事件目标，接收到该事件。沿着DOM树向上冒泡的事件不会触发被指定为use capture（也就是设为true）的listener。当一个元素嵌套了另一个元素，两个元素都对同一个事件注册了一个处理函数时，所发生的事件冒泡和事件捕获是两种不同的事件传播方式。事件传播模式决定了元素以哪个顺序接收事件。进一步的解释可以查看 事件流 及 JavaScript Event order 文档。 如果没有指定， useCapture 默认为 false 。

简单的说该参数决定了元素在哪个阶段触发事件，如果为`true`的时候，元素会在捕获阶段被触发，反之。

### options

> 一个指定有关 listener 属性的可选参数对象。可用的选项如下：
> * capture：`Boolean`，表示listener会在该类型的事件捕获阶段传播到该EventTarget时触发。
> * once：`Boolean`，表示listener在添加之后最多只调用一次。如果是true，listener会在其被调用之后自动移除。
> * passive: `Boolean`，表示listener永远不会调用`preventDefault()`。如果listener仍然调用了这个函数，客户端将会忽略它并抛出一个控制台警告。
> * mozSystemGroup: 只能在 XBL 或者是 Firefox' chrome 使用，这是个 Boolean，表示 listener 被添加到 system group。

这里是我们这次学习的重点。之前我们知道的是第三个参数是一个`Boolean`值，也就是上面介绍的`useCapture`。但是在新版本的DOM规定中，第三个参数由一个简单的布尔值，变为了options对象。

三个属性都是布尔类型的开关，默认值都为 false。其中 capture 属性等价于以前的 useCapture 参数；once 属性就是表明该监听器是一次性的，执行一次后就被自动 removeEventListener 掉，还没有浏览器实现它；passive 属性是本文的主角，Firefox 和 Chrome 已经实现。

其中我们要说一下`passive`。很多移动端的页面都会监听`touchstart`等`touch`事件，像这样：

``` js
document.addEventListener('touchstart', function () {
    ... // 浏览器不知道这里会不会有 e.preventDefault()
})
```

由于`touchstart`事件对象的cancelable属性为true，也就是说它的默认行为可以被监听器通过`preventDefault()` 方法阻止。而它的默认行为通常就是滚动当前页面（还可能是缩放页面），如果它的默认行为被阻止了，页面就必须静止不动。但浏览器无法预先置顶一个监听器会不会调用`preventDefault()`，它能做的只有等监听器执行完之后再去执行默认行为。而监听器执行是要耗时的，有些甚至耗时明显，这样就会导致页面卡顿。即便监听器是个空函数，也会产生一定的卡顿，毕竟空函数的执行也是耗时的。

而实际上有 80% 的滚动事件监听器是不会阻止默认行为的，也就是说大部分情况下，浏览器是白等了。所以`passive`属性的意义就在这里，当它为true的时候，表示浏览器对事件的默认行为是一定执行的，它就可以在两个线程同时执行监听器中的JavaScript代码和浏览器默认行为。

**如果说在一个`passive`的监听器里执行了`preventDefault()`会怎么样？**也没有关系，因为`preventDefault()`不会产生任何效果。同时浏览器会发出警告：`Unable to preventDefault inside passive event listener invocation.`

除了上面在 passive 的监听器里调用 preventDefault() 会发出警告外，Chrome 的开发者工具还会：

1. 发现耗时超过 100 毫秒的非 passive 的监听器，警告你加上 {passive: true}
2. 给监听器对象增加 passive 属性，监听器对象在普通页面中是获取不到的，可以在 Event Listeners 面板中和通过调用 getEventListeners() Command Line API 获取到

**现在该如何调用 removeEventListener？**以前，在第三个参数是布尔值的时候，addEventListener("foo", listener, true) 添加的监听器，必须用 removeEventListener("foo", listener, true) 才能删除掉。因为这个监听器也有可能还注册在了冒泡阶段，那样的话，同一个监听器实际上对应着两个监听器对象（通过 getEventListeners() 可看到）。

那现在 addEventListener("foo", listener, {passive: true}) 添加的监听器该如何删除呢？答案是 removeEventListener("foo", listener) 就可以了，passive 可以省略，原因是：在浏览器内部，用来存储监听器的 map 的 key 是由事件类型，监听器函数，是否捕获这三者组成的，passive 和 once 不在其中，理由显而易见，一个监听器同时是 passive 和非 passive（以及同时是 once 和非 once）是说不通的，如果你添加了两者，那么后添加的不算，浏览器会认为添加过了：

``` js
addEventListener("foo", listener, {passive: true})
addEventListener("foo", listener, {passive: false}) // 这句不算

addEventListener("bar", listener, {passive: false})
addEventListener("bar", listener, {passive: true})  // 这句不算
```

所以说在 removeEventListener 的时候永远不需写 passive 和 once，但 capture 可能要：

``` js
addEventListener("foo", listener, {capture: true})
removeEventListener("foo", listener, {capture: true}) // {capture: true} 必须加，当然 {capture: true} 换成 true 也可以
```

**passive 不能保证什么？**passive 监听器能保证的只有一点，那就是调用 preventDefault() 无效，至于浏览器对默认行为卡顿的优化，那是浏览器的事情，是在规范要求之外的。鉴于这个新特性本来就是为解决滚动和触摸事件的卡顿而发明的，目前 Chrome 和 Firefox 支持优化的事件类型也仅限这类事件，比如 touchstart，touchmove，wheel 等事件。

有几个浏览器不优化的事件类型：

* click 事件
* contextmenu 事件
* beforeinput 事件（只有 Chrome 52 及以上支持

**总结一下就是 {passive: true} 不能保证浏览器对所有事件类型的默认行为进行优化**

除了这三种事件类型外，所有 Cancelable 为 true 的事件类型理论上都是可以有这种优化的、

* [UI 事件类型列表](https://w3c.github.io/uievents/#event-types-list)
* [触摸事件类型列表](https://w3c.github.io/touch-events/#list-of-touchevent-types)

## option支持的安全检测

如果你想检测`passive`，可以如下检测：

``` js
var passiveSupported = false;
try {
    var options = Object.defineProperty({}, "passive", {
        get: function() {
            passiveSupported = true;
        }
    });

    window.addEventListener("test", null, options);
} catch(err) {}
```

然后我们就可以这样使用：

``` js
someElement.addEventListener("mouseup", handleMouseUp, passiveSupported
                               ? { passive: true } : false);
```