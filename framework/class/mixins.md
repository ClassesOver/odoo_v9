# Mixins

该模块定义并暴露出了**ParentedMixin**,  **EventDispatcherMixin**,  **PropertiesMixin** 三个Mixin对象。

**ParentedMixin** 用来构造加入生命周期的特性，每一个对象可有一个父亲对象和多个孩子对象，当一个对象销毁的时候，其孩子也会销毁。

**EventDispatcherMixin** 用于包含事件系统的特性，通过指定目标对象进行注册事件。事件发出对象和目标对象相互存储或引用着，这个用于当其一对象被销毁的时候来移除事件处理器。移除这个引用是必要的来避免内存泄漏和幽灵事件（向之前已经销毁的对象发送事件）。

```js
var EventDispatcherMixin = _.extend({}, ParentedMixin, {
    __eventDispatcherMixin: true,
    init: function() {
        ParentedMixin.init.call(this); // 先调用ParentedMixin的init方法初始化this对象
        this.__edispatcherEvents = new Events(); // 事件调度器，保存对目标对象的引用，真正事件注册和触发的地方
        this.__edispatcherRegisteredEvents = []; // 事件数组，保存对源对象的引用，用于销毁时，移除事件回调函数
    },
    // 注册一个事件
    on: function(events, dest, func) {
        var self = this;
        if (typeof func !== "function") {
            throw new Error("Event handler must be a function.");
        }
        events = events.split(/\s+/); // 以空格分割事件名称，返回一个数组
        _.each(events, function(eventName) {
            self.__edispatcherEvents.on(eventName, func, dest);
            if (dest && dest.__eventDispatcherMixin) { 
                // 如果目标对象也是拥有EventDispatcherMixin特性，使用__edispatcherRegisteredEvents这个数组保存对源对象的引用
                dest.__edispatcherRegisteredEvents.push({name: eventName, func: func, source: self});
            }
        });
        return this;
    },
    // 取消注册一个事件
    off: function(events, dest, func) {
        var self = this;
        events = events.split(/\s+/); // 以空格分割事件名称，返回一个数组
        _.each(events, function(eventName) {
            self.__edispatcherEvents.off(eventName, func, dest);
            if (dest && dest.__eventDispatcherMixin) { 
              // 如果目标对象也是拥有EventDispatcherMixin特性，也将从__edispatcherRegisteredEvents移除源对象的引用
                dest.__edispatcherRegisteredEvents = _.filter(dest.__edispatcherRegisteredEvents, function(el) {
                    return !(el.name === eventName && el.func === func && el.source === self);
                });
            }
        });
        return this;
    },
    trigger: function() {
        this.__edispatcherEvents.trigger.apply(this.__edispatcherEvents, arguments);
        return this;
    },
    trigger_up: function(name, info) {
        var event = new OdooEvent(this, name, info);
        this._trigger_up(event);
    },
    _trigger_up: function(event) {
        var parent;
        this.__edispatcherEvents.trigger(event.name, event);
        if (!event.is_stopped() && (parent = this.getParent())) {
            parent._trigger_up(event);
        }
    },
    destroy: function() {
        var self = this;
        // 当前目标对象销毁的时候，通过保留的事件引用数组来迭代源对象来移除对象的事件回调函数
        _.each(this.__edispatcherRegisteredEvents, function(event) {
            event.source.__edispatcherEvents.off(event.name, event.func, self);
        });
        this.__edispatcherRegisteredEvents = [];
        // 也移除当前目标对象自身维护的事件注册的回调函数
        _.each(this.__edispatcherEvents.callbackList(), function(cal) {
            this.off(cal[0], cal[2], cal[1]);
        }, this);
        this.__edispatcherEvents.off();
        ParentedMixin.destroy.call(this);
    }
    });
```

备注：

`/\s+/ `该正则对象匹配多个空白字符。

事件真正注册调用和触发的地方 **Events**，处理事件的分发，有三个重要的函数：on 、off 、trigger，不要直接使用和继承，相反使用 **EventDispatcherMixin**。

* `on(events, callback, context)`

```js
    on : function(events, callback, context) {
        var ev;
        events = events.split(/\s+/);
        var calls = this._callbacks || (this._callbacks = {});
        while ((ev = events.shift())) {
            var list = calls[ev] || (calls[ev] = {});
            var tail = list.tail || (list.tail = list.next = {});
            tail.callback = callback;
            tail.context = context;
            list.tail = tail.next = {};
        }
        return this;
   }
```

\_classbacks 这个内部对象为每个注册的事件维护一个单向调用链表，每当注册一个回调函数，就会在尾部节点加入callback和context\(dest\)属性然后在尾部新增加一个节点，具体下图所示：

![](/assets/event_callbacks.jpg)

* `off(events, callback, context)`

```js
    off : function(events, callback, context) {
        var ev, calls, node;
        if (!events) {
            delete this._callbacks;
        } else if ((calls = this._callbacks)) {
            events = events.split(/\s+/);
            while ((ev = events.shift())) {
                node = calls[ev];
                delete calls[ev];
                if (!callback || !node)
                    continue;
                while ((node = node.next) && node.next) {
                    if (node.callback === callback
                            && (!context || node.context === context))
                        continue;
                    this.on(ev, node.callback, node.context);
                }
            }
        }
        return this;
    }
```

每次off的时候，都会删除\_callbacks，然后除了off的事件回调外会重新注册所有回调。

