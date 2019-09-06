* JavaScript 只有数字类型，内部被表示为 64 为的浮点数，不区分整型和浮点型，所以 1 和 1.0 是等价的
* JavaScript 中没有字符类型，如果你想要表示字符，那么只要创建一个包含一个字符的字符串即可
* JavaScript 除了数字、字符串、布尔值、null 和 undefined 值，其他的都是对象类型



## Object.create 实现

```
if (typeof Object.create !== "function") {
    Object.create = function (proto, propertiesObject) {
    	  // 参数正确性检查
        if (typeof proto !== 'object' && typeof proto !== 'function') {
            throw new TypeError('Object prototype may only be an Object: ' + proto);
        } else if (proto === null) {
            throw new Error("This browser's implementation of Object.create is a shim and doesn't support 'null' as the first argument.");
        }
        if (typeof propertiesObject != 'undefined') throw new Error("This browser's implementation of Object.create is a shim and doesn't support a second argument.");

		 // 原型继承
        function F() {}
        F.prototype = proto;
        return new F();
    };
}
```


## this的值和Funcion的四种调用方式

1. 方法调用 obj.func()，this 就是 obj
2. 函数调用，this 就是 js 的全局环境变量
3. 构造函数调用 new Func()
4. apply、call 的绑定调用


**new 运算符的实现原理模拟**

```
Function.method('new', function(){
    // 创建一个新对象，原型是函数 的 prototype
    var that = Object.create(this.prototype)
    // 调用构造器函数，绑定 -this- 到新对象上
    var other = this.apply(that, arguments)
    // 如果它返回值不是一个对象，就返回该新对象
    return (typeof other === 'object' && other) || that
})

```






