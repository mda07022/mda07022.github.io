---
layout: post
title: 分析John Resig's "Simple JavaScript Inheritance"代码
tags: [js]
---


### 代码
{% highlight js linenos %}
(function(){
    var initializing = false, fnTest = /xyz/.test(function(){xyz;}) ? /\b_super\b/ : /.*/;
    this.Class = function(){};
    Class.extend = function(prop) {
        var _super = this.prototype;
        initializing = true;
        var prototype = new this();
        initializing = false;
        for (var name in prop) {
            prototype[name] = typeof prop[name] == "function" &&
                typeof _super[name] == "function" && fnTest.test(prop[name]) ?
                (function(name, fn){
                    return function() {
                        var tmp = this._super;
                        this._super = _super[name];
                        var ret = fn.apply(this, arguments);
                        this._super = tmp;
                        return ret;
                    };
                })(name, prop[name]) :
                prop[name];
        }
        function Class() {
            if ( !initializing && this.init )
                this.init.apply(this, arguments);
        }
        Class.prototype = prototype;
        Class.constructor = Class;
        Class.extend = arguments.callee;
        return Class;
    };
})();
{% endhighlight %}


### 代码分析

首先(function(){})()创建一个匿名的自执行函数，作为作用域，与外部隔离。

initializing作为开关判断第25行是否执行init方法，避免在不必要的情况下执行init方法。如果extend方法的调用者存在init方法，且initializing为false则执行,否则不执行。

{% highlight js linenos %} fnTest = /xyz/.test(function(){xyz;}) ? /\b_super\b/ : /.*/; {% endhighlight %}
这个fnTest的目的就是为了验证 class method 中是否使用了 "_super()" 调用. 这种技术叫做 " function decompilation(函数反编译)" 
也叫做 "function serialisation(函数序列化)"， Function serialisation 是在一个函数被转换成字符串时发生的. 现在很多浏览器都支持 toString 方法。

如果function(){xyz;}转成字符串成功则返回xyz, /xyz/.test(function(){xyz;})为true,则表达式结果为\b_super\b，否则为/.**/（即对所有字符串都判断）。
浏览器不支持 Function serialisation 将会始终返回 true, 那么会始终对 _super 进行处理，保证所有浏览器的可用。

第2行代码this.Class = function(){};定义一个空的构造函数。this为windows。

第3行代码Class.extend = function(prop) 为window.Class构造函数添加一个名为extend的方法，window.Class的原型(prototype)为Object

{% highlight js linenos %} var _super = this.prototype; {% endhighlight %}
第5行代码将**当前对象**的原型存储在 _super中,this代表调用extend方法的对象, this.prototype是被扩展对象的原型, 它可以访问父对象。

{% highlight js linenos %}
initializing = true;
var prototype = new this();
initializing = false;
{% endhighlight %}
上述代码将**当前对象**的实例的引用赋值给prototype变量，initializing = true避免执行init方法

for (var name in prop) 遍历prop的所有属性，并赋值给prototype变量。如果当前属性满足3个条件1.是function， 2._super中同名属性也是function， 3.fnTest.test(prop[name])为true
则对该方法进行处理。

{% highlight js linenos %}
function(name, fn){
    return function() {
        var tmp = this._super;
        this._super = _super[name];
        var ret = fn.apply(this, arguments);
        this._super = tmp;
        return ret;
    };
})(name, prop[name])
{% endhighlight %}
如果满足上面3个条件则用一个自执行匿名函数，返回一个匿名方法替代prop[name]代表的方法。先将**当前对象**的_super属性保存到临时变量tmp，如果当前对象是windows.Class则this._super为undefined,
将**当前对象**的_super属性赋值为_super\[name](_super在第5行定义为当前对象的原型)，这样执行的时候_super就替换为**当前对象**原型（父类）中的同名方法。
 这样当 fn 通过 apply 被执行的时候 this._super()就会指向 父类方法, 这个父类方法中的 this动态指向当前对象.
然后var ret = fn.apply(this, arguments);当前对象对fn方法调用的引用指向ret变量相当于 var ret = this.fn(arguments)。

{% highlight js linenos %}
function Class() {
    if ( !initializing && this.init )
        this.init.apply(this, arguments);
}
{% endhighlight %}
这里定义了一个Class构造函数，与之前的windows.Class是不同的对象，这一点很重要。

{% highlight js linenos %}
Class.prototype = prototype;
Class.constructor = Class;
Class.extend = arguments.callee;
return Class;
{% endhighlight %}
上面的代码经过遍历，把prop对象属性转移过去之后的prototype变量赋值给刚刚定义的构造方法的prototype属性。

定义constructor方法为Class它自己。

Class.extend = arguments.callee; 给Class构造方法定义一个extend方法，这个方法通过arguments.callee将赋extend自身，返回在第3行定义的windows.Class.extend方法。
最后返回上面代码第一行定义的Class构造方法。

所有最后返回的是一个新的构造方法，它的prototype(父级)是**当前对象**（第7行new this），他的constructor是它本身，它有一个extend方法(第4行定义的)，这样就实现了继承。


### 例子
下面有个简单的例子,  定义个简单的 Foo继承windows.Class ， 创建继承对象 Bar:

{% highlight js linenos %}
var Foo = Class.extend({
    qux: function() {
        return "Foo.qux";
    }
});
var Bar = Foo.extend({
    qux: function() {
        return "Bar.qux, " + this._super();
    }
});
var bar = new Bar();
{% endhighlight %}

var Foo = Class.extend返回构造方法Class,可以通过构造方法Class实例化，原型是windows.Class
Foo.extend返回新的构造方法Class,原型是Foo,Foo的原型又是windows.Class
var bar = new Bar();通过构造方法Class实例化，执行第23行定义的构造方法实例化。

###参考资料：
> [1.全面理解面向对象的 JavaScript](http://www.ibm.com/developerworks/cn/web/1304_zengyz_jsoo/)

> [2.理解John Resig's 'Simple JavaScript Inheritance'代码](http://www.cnblogs.com/enein/archive/2012/12/03/2799160.html)

