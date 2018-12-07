# Boot

该文件定义了Odoo的模块系统，需要首先被加载，整体结构是一个立即执行函数，通过window全局对象来暴露在函数定义的私有内部变量`var odoo = window.odoo`。

**属性**

* **jobs**

加载模块任务单元的数组容器，job就是模块载入的任务单元，具体的 job 结构如下：

```
{
    name: name, // moduleName
    factory: factory, // function(require)
    deps: deps, // dependencies
}
```

odoo的define定义的过程可理解为解析成任务单元，并压入数组容器的过程。

* **job\_names**

任务单元名称数组容器，维护就是定义各个模块的名字`moduleName`。

* **job\_deps**

任务单元依赖数组容器，具体结构如下:

```
{  
   from:dep,
   to:moduleName
}
```

* **job\_deferred**

* **factories**

模块定义的对象容器，基于源码的字面意思，我们下文所称的模块工厂函数就是我们的模块定义函数。

```js
 var factories = Object.create(null);
```

这里创建了没有继承关系的空对象，没有上层Object原型，仅仅是一个维护着具体各个模块定义函数对象属性`{moduleName:  factory}`，下面可以看到具体的结构。

![](/assets/boot_factories.png)

* **services**

所谓的服务就是我们模块所暴露的对外的接口\(可以require\)，从代码上将就是模块定义函数执行返回的结果，我们实际require的就是从services对象来pick出我们所需的依赖。

```
var services = Object.create({
    qweb: new QWeb2.Engine(),
    $: $,
    _: _,
});
```

这里以`{ qweb: new QWeb2.Engine(), $: $, _: _,}`原型创建了services，姑且简单点说就是继承这个对象，可直接require。

我们看下require的真面目：

```
    function make_require (job) {
        var deps = _.pick(services, job.deps);

        function require (name) {
            if (!(name in deps)) {
                console.error('Undefined dependency: ', name);
            } else {
                require.__require_calls++;
            }
            return deps[name];
        }

        require.__require_calls = 0;
        return require;
    }
```

备注：

`_.pick(object, *keys)`该函数可以选择属性完成services对象的拷贝。

![](/assets/boot_pick_services.png)

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

`_.uniqueId([prefix])`underscore 生成以prefix为前缀的唯一Id。

