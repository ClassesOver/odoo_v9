# Boot

该文件定义了Odoo的模块系统，需要首先被加载，整体结构是一个立即执行函数，通过window全局对象来暴露在函数定义的私有内部变量`var odoo = window.odoo`。

**属性**

* jobs

数组

* job\_names

* job\_deps

* job\_deferred

* factories

```js
Object.create(null);
```

这里创建了继承关系的空对象，没有上层原型，只是一个维护着odoo的具体各个模块定义函数的对象。

![](/assets/boot_factories.png)

**函数**

* `define(moduleName,dependencies,func)`

_**moduleName  **_参数是javascript模块的名字，这里有个规约必须应该保证唯一性，遵循特定的描述，例如“web.Widget”这个意味着定义odoo的web模块\(addon\)下，可导出Widget类（因为这里首字母大写）。

_**dependencies** _参数是可选的，这个是一个数组来声明在模块执行前需要加在的依赖模块，如果没有显式指定，将会从具体工厂函数也就是第三个参数来获取所需依赖，通过toString方法转化为字符串，通过正则表达式来抽取。

_**func  **_这个参数具体定义这个模块，返回的值就是模块的值，javascript存在全局和函数作用域，无块级作用域\(ES 6新增块级\)，因此通过函数作用域达到模块命名空间的效果。

```
        define: function () {
            // 将 arguments 类数组对象转化为数组对象，转化为数组就可以利用数组的便利方法
            var args = Array.prototype.slice.call(arguments); 
            // 判断第一个参数是否为字符串
            var name = typeof args[0] === 'string' ? args.shift() : _.uniqueId('__job'); 
            // 获取最后一个参数也就我们定义的function
            var factory = args[args.length - 1];
            var deps;
            // 这里是对模块依赖的判断，若第二参数定义了依赖就不会从工厂函数中抽取所需的依赖。
            if (args[0] instanceof Array) {
                deps = args[0];
            } else {
                deps = [];
                factory.toString()
                    .replace(commentRegExp, '')
                    .replace(cjsRequireRegExp, function (match, dep) {
                        deps.push(dep);
                    });
            }
```

备注: 

`Array.prototype.shift()` 内建数组方法，移除并返回数组的第一个元素。

`_.uniqueId([prefix]) `underscore 生成以prefix为前缀的唯一Id。

