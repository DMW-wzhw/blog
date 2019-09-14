# 阅读 JSPatch 源码 01

首先是阅读源码的 js 部分

## JSPatch.js


```
var global = this

;(function() {
  // 用于保存下发的 JS 代码中定义的类和类的实例和类方法
  // 因为在下方的 JS 代码中，方法中调用的方法是 JS 中写的，那么就可以直接找到，而不需要经过 OC 这一层转发
  /**
   defineClass('JPViewController', {
      handleBtn: function(sender) {
        self.test()
        //self.__c("test")()
      },
      test: function() {
        console.log('dddd')
        //console.__c("log")('ddddd')
    }
  })
   */
  var _ocCls = {};
  var _jsCls = {};

  var _formatOCToJS = function(obj) {
    if (obj === undefined || obj === null) return false
    if (typeof obj == "object") {
      if (obj.__obj) return obj
      // 这里可以判断 OC 调用后是否是返回了 nil, 如果是 nil，那么就用 boolean 代替，为了链式调用
      // https://github.com/bang590/JSPatch/wiki/添加-struct-类型支持
      // OC nil => JS {__isNil: true}
      if (obj.__isNil) return false
    }
    if (obj instanceof Array) {
      var ret = []
      obj.forEach(function(o) {
        ret.push(_formatOCToJS(o))
      })
      return ret
    }
    // OC 的 block 对象
    if (obj instanceof Function) {
        return function() {
            var args = Array.prototype.slice.call(arguments)
            var formatedArgs = _OC_formatJSToOC(args)
            for (var i = 0; i < args.length; i++) {
                if (args[i] === null || args[i] === undefined || args[i] === false) {
                formatedArgs.splice(i, 1, undefined)
            } else if (args[i] == nsnull) {
                formatedArgs.splice(i, 1, null)
            }
        }
        return _OC_formatOCToJS(obj.apply(obj, formatedArgs))
      }
    }
    if (obj instanceof Object) {
      var ret = {}
      for (var key in obj) {
        ret[key] = _formatOCToJS(obj[key])
      }
      return ret
    }
    // 其他的 OC 对象，那么使用的就是该对象的指针
    return obj
  }
  
  var _methodFunc = function(instance, clsName, methodName, args, isSuper, isPerformSelector) {
    var selectorName = methodName
    if (!isPerformSelector) {
      methodName = methodName.replace(/__/g, "-")
      selectorName = methodName.replace(/_/g, ":").replace(/-/g, "_")
      var marchArr = selectorName.match(/:/g)
      var numOfArgs = marchArr ? marchArr.length : 0
      if (args.length > numOfArgs) {
        selectorName += ":"
      }
    }
    var ret = instance ? _OC_callI(instance, selectorName, args, isSuper):
                         _OC_callC(clsName, selectorName, args)
    return _formatOCToJS(ret)
  }

  var _customMethods = {
    // 元函数 UIView.alloc().init() 通过正则转为 UIView.__c('alloc')().__init('init')()
    // UIView.__c('alloc') 返回一个函数后，UIView.__c('alloc')() 调用
    // 以 superView.__c("addSubview")(view) 为例子
    /*
    首先是 superView.__c("addSubview") 返回个 function
    slf: {
      __obj: superView,
      __clsName: UIView
    }
    return function(){
        var args = Array.prototype.slice.call(arguments)
        return _methodFunc(slf.__obj, slf.__clsName, methodName, args, slf.__isSuper)
    }
    然后在 superView.__c("addSubview")(view) 调用这个function
    => _methodFunc(superView, UIView, 'addSubview', [view], false)
    */
    __c: function(methodName) {
      var slf = this

      if (slf instanceof Boolean) {
        // 因为在 jspatch 中使用 false 来代表 nil
        // nil 调用任何返回还是返回 nil
        return function() {
          return false
        }
      }
      if (slf[methodName]) {
        return slf[methodName].bind(slf);
      }

      if (!slf.__obj && !slf.__clsName) {
        throw new Error(slf + '.' + methodName + ' is undefined')
      }
      if (slf.__isSuper && slf.__clsName) {
          slf.__clsName = _OC_superClsName(slf.__obj.__realClsName ? slf.__obj.__realClsName: slf.__clsName);
      }
      var clsName = slf.__clsName
      if (clsName && _ocCls[clsName]) {
        var methodType = slf.__obj ? 'instMethods': 'clsMethods'
        if (_ocCls[clsName][methodType][methodName]) {
          slf.__isSuper = 0;
          return _ocCls[clsName][methodType][methodName].bind(slf)
        }
      }

      return function(){
        var args = Array.prototype.slice.call(arguments)
        return _methodFunc(slf.__obj, slf.__clsName, methodName, args, slf.__isSuper)
      }
    },
    
    // 为 slf 对象添加一个 isSuper 表示
    // self.super()
    /*
    slf: {
      __obj: superView,
      __clsName: UIView
    }
    =>
    slf: {
      __obj: obj,
      __clsName: XXX,
      __isSuper: 1
    }    
    */
    super: function() {
      var slf = this
      if (slf.__obj) {
        slf.__obj.__realClsName = slf.__realClsName;
      }
      return {__obj: slf.__obj, __clsName: slf.__clsName, __isSuper: 1}
    },

    // https://github.com/bang590/JSPatch/wiki/performSelectorInOC-使用文档
    // 查看 JPEngine 的 JPForwardInvocation 大概在 808 行
    // 原理是在 OC JPForwardInvocation 拿到返回值后，判断返回值对象是否有 __isPerformInOC 属性，并且值为 1
    // 如果有那么做特殊处理
    // 例如 sel 对应的方法在 js 执行很耗时，那么可以转到 OC 中执行，执行完后，在将结果作为参数给 cb，在执行 cb
    // jsval = [cb callWithArguments:args];
    // 在 OC 中执行的方法不会阻碍 js 的执行线程
    performSelectorInOC: function() {
      var slf = this
      var args = Array.prototype.slice.call(arguments)
      return {__isPerformInOC:1, obj:slf.__obj, clsName:slf.__clsName, sel: args[0], args: args[1], cb: args[2]}
    },

    // js 的 performSelector 方法
    /**
     superView.performSelector('addSubview:', view)
     */
    performSelector: function() {
      var slf = this
      var args = Array.prototype.slice.call(arguments)
      return _methodFunc(slf.__obj, slf.__clsName, args[0], args.splice(1), slf.__isSuper, true)
    }
  }
  // 将上面的方法添加到 Object.prototype 上
  for (var method in _customMethods) {
    if (_customMethods.hasOwnProperty(method)) {
      Object.defineProperty(Object.prototype, method, {value: _customMethods[method], configurable:false, enumerable: false})
    }
  }

  // 导入 OC 库，为了能使用 UIView 这些关键字
  // 以 UIView 为🌰
  /**
   gloable.UIView = {
     __clsName: UIView
   }
   */
  var _require = function(clsName) {
    if (!global[clsName]) {
      global[clsName] = {
        __clsName: clsName
      }
    } 
    return global[clsName]
  }

  global.require = function() {
    var lastRequire
    for (var i = 0; i < arguments.length; i ++) {
      arguments[i].split(',').forEach(function(clsName) {
        lastRequire = _require(clsName.trim())
      })
    }
    return lastRequire
  }

  var _formatDefineMethods = function(methods, newMethods, realClsName) {
    // 遍历要添加的 JS 方法
    for (var methodName in methods) {
      // methodName => key
      // 如果不是 JS function 则返回
      if (!(methods[methodName] instanceof Function)) return;
      // 执行下面方法
      (function(){
        // 获取 key(methodName) 对应的 function
        var originMethod = methods[methodName]

        // 把 JSFunction 保存到 newMethods 中，为了后续将其保持到 __clsOC 对象中，方便在 js 代码中调用 js 函数的时候
        // 不需要在走 OC 那一层
        /*
        {
          方法名1: [参数个数, 方法的 JS 实现],
          方法名2: [参数个数, 方法的 JS 实现],
        }
        */
        newMethods[methodName] = [originMethod.length, function() {
          try {
            // 参数的第一个为是 self，是怎么传递进来的？
            // 答案在 JSEngine 中的 JPForwardInvocation 方法中，会将对象最为第一个参数传递进来
            // // 828 行
            var args = _formatOCToJS(Array.prototype.slice.call(arguments))
            var lastSelf = global.self
            global.self = args[0]
            if (global.self) global.self.__realClsName = realClsName
            args.splice(0,1)
            var ret = originMethod.apply(originMethod, args)
            global.self = lastSelf
            return ret
          } catch(e) {
            _OC_catch(e.message, e.stack)
          }
        }]
      })()
    }
  }

  // 对于 JS 的方法封装 self 关键字
  var _wrapLocalMethod = function(methodName, func, realClsName) {
    return function() {
      var lastSelf = global.self
      global.self = this
      this.__realClsName = realClsName
      var ret = func.apply(this, arguments)
      global.self = lastSelf
      return ret
    }
  }

  var _setupJSMethod = function(className, methods, isInst, realClsName) {
    for (var name in methods) {
      var key = isInst ? 'instMethods': 'clsMethods',
          func = methods[name]
      // 包装 self 到参数中去
      _ocCls[className][key][name] = _wrapLocalMethod(name, func, realClsName)
    }
  }

  var _propertiesGetFun = function(name){
    return function(){
      var slf = this;
      if (!slf.__ocProps) {
        var props = _OC_getCustomProps(slf.__obj)
        if (!props) {
          props = {}
          _OC_setCustomProps(slf.__obj, props)
        }
        slf.__ocProps = props;
      }
      return slf.__ocProps[name];
    };
  }

  var _propertiesSetFun = function(name){
    return function(jval){
      var slf = this;
      if (!slf.__ocProps) {
        var props = _OC_getCustomProps(slf.__obj)
        if (!props) {
          props = {}
          _OC_setCustomProps(slf.__obj, props)
        }
        slf.__ocProps = props;
      }
      slf.__ocProps[name] = jval;
    };
  }

  global.defineClass = function(declaration, properties, instMethods, clsMethods) {
    var newInstMethods = {}, newClsMethods = {}
    if (!(properties instanceof Array)) {
      clsMethods = instMethods
      instMethods = properties
      properties = null
    }

    // 设置属性的 _OC_getCustomProps 和 _OC_setCustomProps 方法
    if (properties) {
      properties.forEach(function(name){
        if (!instMethods[name]) {
          instMethods[name] = _propertiesGetFun(name);
        }
        var nameOfSet = "set"+ name.substr(0,1).toUpperCase() + name.substr(1);
        if (!instMethods[nameOfSet]) {
          instMethods[nameOfSet] = _propertiesSetFun(name);
        }
      });
    }
    // 获取类名，因为这里的 declaration 一般是 类名 : 父类 (协议) 这种格式
    var realClsName = declaration.split(':')[0].trim()

    // 将实例方法保存到 newInstMethods 中
    _formatDefineMethods(instMethods, newInstMethods, realClsName)
    // 将类方法保存到 newClsMethods 中
    _formatDefineMethods(clsMethods, newClsMethods, realClsName)

    // 调用 OC 中定义的方法，为 类 添加实例方法和类方法
    // ret 是返回的类名
    var ret = _OC_defineClass(declaration, newInstMethods, newClsMethods)
    var className = ret['cls']
    var superCls = ret['superCls']
    
    // 将 js 方法保存起来
    /**
     _ocCls = {
      className: {
        instMethods: {
          func1: 
        },
        clsMethods: {

        }
      }
     }
     */
    _ocCls[className] = {
      instMethods: {},
      clsMethods: {},
    }

    if (superCls.length && _ocCls[superCls]) {
      for (var funcName in _ocCls[superCls]['instMethods']) {
        _ocCls[className]['instMethods'][funcName] = _ocCls[superCls]['instMethods'][funcName]
      }
      for (var funcName in _ocCls[superCls]['clsMethods']) {
        _ocCls[className]['clsMethods'][funcName] = _ocCls[superCls]['clsMethods'][funcName]
      }
    }

    _setupJSMethod(className, instMethods, 1, realClsName)
    _setupJSMethod(className, clsMethods, 0, realClsName)

    return require(className)
  }

  global.defineProtocol = function(declaration, instProtos , clsProtos) {
      var ret = _OC_defineProtocol(declaration, instProtos,clsProtos);
      return ret
  }

  global.block = function(args, cb) {
    var that = this
    var slf = global.self
    if (args instanceof Function) {
      cb = args
      args = ''
    }
    var callback = function() {
      var args = Array.prototype.slice.call(arguments)
      global.self = slf
      return cb.apply(that, _formatOCToJS(args))
    }
    var ret = {args: args, cb: callback, argCount: cb.length, __isBlock: 1}
    if (global.__genBlock) {
      ret['blockObj'] = global.__genBlock(args, cb)
    }
    return ret
  }
  
  if (global.console) {
    var jsLogger = console.log;
    global.console.log = function() {
      global._OC_log.apply(global, arguments);
      if (jsLogger) {
        jsLogger.apply(global.console, arguments);
      }
    }
  } else {
    global.console = {
      log: global._OC_log
    }
  }

  global.defineJSClass = function(declaration, instMethods, clsMethods) {
    var o = function() {},
        a = declaration.split(':'),
        clsName = a[0].trim(),
        superClsName = a[1] ? a[1].trim() : null
    o.prototype = {
      init: function() {
        if (this.super()) this.super().init()
        return this;
      },
      super: function() {
        return superClsName ? _jsCls[superClsName].prototype : null
      }
    }
    var cls = {
      alloc: function() {
        return new o;
      }
    }
    for (var methodName in instMethods) {
      o.prototype[methodName] = instMethods[methodName];
    }
    for (var methodName in clsMethods) {
      cls[methodName] = clsMethods[methodName];
    }
    global[clsName] = cls
    _jsCls[clsName] = o
  }
  
  global.YES = 1
  global.NO = 0
  global.nsnull = _OC_null
  global._formatOCToJS = _formatOCToJS
  
})()

```

