# Class

实现的Javascript的类继承，定义了`OdooClass`构造函数，用于生成`Class`的原型对象，`extend`函数用于初次（初始化）构建扩展我们的生成`Class`原型对象，`include`用于修改已经存在的目标原型对象\(`this._super`在具体目标函数执行的时候存放之前的原型同名函数属性\)。

```js
function OdooClass(){};
```

该文件重要的函数就是`extend`,这个是OdooClass的静态方法，用来生成类，因为Javascript仅支持原型的单继承，通过mixin的方式来实现多继承，mixin简单通俗的讲就是把一个对象的方法和属性拷贝到另一个对象上。odoo的`extend`就是mixin的实现具体方法，下面代码描述了`extend`是如何构造原型的：

```js
    var _super = this.prototype; //指向 OdooClass.prototype
    // Support mixins arguments
    var args = _.toArray(arguments);
    args.unshift({});
    var prop = _.extend.apply(_,args); // 利用apply,将参数以数组的方式传入，无需关心参数的个数

    // Instantiate a web class (but only create the instance,
    // don't run the init constructor)
    initializing = true;
    var This = this; // 指向function OdooClass
    var prototype = new This(); // new OdooClass() 生成OdooClass{} 对象，作为function Class的原型
    initializing = false;

    // Copy the properties over onto the new prototype
    // 将属性复制到上面生成的原型上
    _.each(prop, function(val, name) {
        // Check if we're overwriting an existing function
        prototype[name] = typeof prop[name] == "function" &&
                          fnTest.test(prop[name]) ?
                (function(name, fn) {
                    return function() {
                        var tmp = this._super;

                        // Add a new ._super() method that is the same
                        // method but on the super-class
                        this._super = _super[name];

                        // The method only need to be bound temporarily, so
                        // we remove it when we're done executing
                        var ret = fn.apply(this, arguments);
                        this._super = tmp;

                        return ret;
                    };
                })(name, prop[name]) :
                prop[name];
    });
```

Class的继承关系图如下：  
![](/assets/class.jpg)  
Class的extend和include静态方法关系图如下：  
![](/assets/class_extend_include.jpg)  
备注:

`Array.prototype.unshift(element,...)`用来向头部添加新的元素，不产生新的数组。

`_.extend(destination, *sources)` 将source对象中的所有属性简单地覆盖到destination对象上。

此外，还有一个比较刚开始没看懂的一个函数`fnTest`：

```js
var fnTest = /xyz/.test(function(){xyz();}) ? /\b_super\b/ : /.*/;
```

该函数在`include`和`extend` 函数中都有用到，在给原型添加新增同名函数成员的时候，该函数会判断这个成员函数是否函数`_super的调用`。  
在`extend`函数中，若存在就是在函数执行的时候，将原始的原型同名`_super[name]`函数在执行时绑定到`this._super`上，根据代码可知就是原始的`this.prototype`，这个`this`可能是`OdooClass`也可能是`Class`，两者都有`extend`方法：

```js
    var tmp = this._super;

        // Add a new ._super() method that is the same
        // method but on the super-class
    this._super = _super[name];

    // The method only need to be bound temporarily, so
    // we remove it when we're done executing
    var ret = fn.apply(this, arguments);
    this._super = tmp;
```

在'include'函数中，与`extend`函数有些不同，`include `会修改创建原型对象，在`prototype[name]`被重新赋值前，`this._super`会绑定之前的`prototype[name]`：

```js
    prototype[name] = (function (name, fn, previous) {
        return function () {
            var tmp = this._super;
            this._super = previous;
            var ret = fn.apply(this, arguments);
            this._super = tmp;
            return ret;
        };
    })(name, properties[name], prototype[name]);
```



