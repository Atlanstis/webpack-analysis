# 懒加载

## 懒加载流程

1. Import() 可以实现指定模块的懒加载操作。
2. 懒加载的核心原理是 jsonp。
3. t 方法可以针对内容进行不同的处理。（根据传入的数值）

### t 方法分析

用于加载指定 value 的模块内容，之后对内容进行处理再返回。

```javascript
__webpack_require__.t = function (value, mode) {
  /**
   * 01 接受两个参数，一个是 value，一般用于表示被加载的模块id，第二个值 mode 是一个二进制的数值
   * 02 t 方法内部做的第一件事情就是调用自定义的 require 方法，加载 value 对应的模块导出，重新赋值给 value
   * 03 当获取到了这个 value 值之后，余下的 8 4 ns 2 都是对当前的内容进行加工处理，然后返回使用
   * 04 当 mode & 8 成立时，直接将 value 返回（懒加载 commonjs 规范导出的内容）
   * 05 当 mode & 4 成立时，将 value 返回（处理 esModule 规范的内容）
   * 06 如果上述条件不成立，还是要继续处理 value，定义一个 ns {}
   * 06-1 如果拿到的 value 是一个可以直接使用的内容，例如是一个字符串，将它挂载到 ns 的 default 属性上
   * 06-2 如果拿到的 value 是一个对象，给 ns 上添加每个键值相对应的 getter 方法，返回相对应的值
   */
  if (mode & 1) value = __webpack_require__(value)
  if (mode & 8) return value
  if (mode & 4 && typeof value === 'object' && value && value.__esModule)
    return value
  var ns = Object.create(null)
  __webpack_require__.r(ns)
  Object.defineProperty(ns, 'default', { enumerable: true, value: value })
  if (mode & 2 && typeof value != 'string')
    for (var key in value)
      __webpack_require__.d(
        ns,
        key,
        function (key) {
          return value[key]
        }.bind(null, key)
      )
  return ns
}
```

### 懒加载 import

#### 懒加载转换

```javascript
import(/*webpackChunkName: "login"*/ './login.js').then((login) => {
  console.log(login)
})
```

转换后格式:

```javascript
__webpack_require__
  .e(/*! import() | login */ 'login')
  .then(__webpack_require__.t.bind(null, /*! ./login.js */ './src/login.js', 7))
  .then((login) => {
    console.log(login)
  })
```

#### 被懒加载模块转换

```javascript
module.exports = '懒加载导出内容'
```

转换后格式:

```javascript
;(window['webpackJsonp'] = window['webpackJsonp'] || []).push([
  ['login'],
  {
    './src/login.js':
      /*! no static exports found */
      function (module, exports) {
        module.exports = '懒加载导出内容'
      }
  }
])
```

## 懒加载 代码

```javascript
;(function (modules) {
  // 4 webpackJsonpCallback作用：合并模块定义、改变 Promise 状态，执行后续行为
  function webpackJsonpCallback(data) {
    // 01 获取需要被动态加载的模块 id
    var chunkIds = data[0]
    // 02 获取需要被加载加载的模块依赖关系对象
    var moreModules = data[1]

    var moduleId,
      chunkId,
      i = 0,
      resolves = []
    // 03 循环判断 chunkIds 里对应的模块内容是否已经完成了加载
    for (; i < chunkIds.length; i++) {
      chunkId = chunkIds[i]
      if (
        Object.prototype.hasOwnProperty.call(installedChunks, chunkId) &&
        installedChunks[chunkId]
      ) {
        // 正在加载中，第 0 项对应 promise 的 resolve 方法
        resolves.push(installedChunks[chunkId][0])
      }
      // 更新当前的 chunk 状态 
      installedChunks[chunkId] = 0
    }
    // 合并模块定义
    for (moduleId in moreModules) {
      if (Object.prototype.hasOwnProperty.call(moreModules, moduleId)) {
        modules[moduleId] = moreModules[moduleId]
      }
    }
    if (parentJsonpFunction) parentJsonpFunction(data)
		
    // 触发 resolve
    while (resolves.length) {
      resolves.shift()()
    }
  }

  var installedModules = {}

  /**
   * 5 定义 installedModules 用于标识某个被懒加载的 chunkId 对应的 chunk 是否完成了加载
   * 0：已经被加载
   * Promise：正在加载中
   * undefined：未被加载
   * null：未被加载，预加载中（null = chunk preloaded/prefetched）
   */
  var installedChunks = {
    main: 0
  }

  // script path function
  function jsonpScriptSrc(chunkId) {
    return __webpack_require__.p + '' + chunkId + '.built.js'
  }

  // The require function
  function __webpack_require__(moduleId) {
    // Check if module is in cache
    if (installedModules[moduleId]) {
      return installedModules[moduleId].exports
    }
    // Create a new module (and put it into the cache)
    var module = (installedModules[moduleId] = {
      i: moduleId,
      l: false,
      exports: {}
    })

    // Execute the module function
    modules[moduleId].call(
      module.exports,
      module,
      module.exports,
      __webpack_require__
    )

    // Flag the module as loaded
    module.l = true

    // Return the exports of the module
    return module.exports
  }

  // 6 实现：实现 jsonp 来加载内容；利用 promise 来实现异步加载操作
  __webpack_require__.e = function requireEnsure(chunkId) {
    // 01 定义一个 promise 数组用于存放 promise
    var promises = []

		// 02 获取 chunkId 对应的 chunk 是否已经加载
    var installedChunkData = installedChunks[chunkId]
    // 03 依据当前是否完成加载的状态执行后续的逻辑
    if (installedChunkData !== 0) {
      // 0：代表已经加载，此处代表还未加载
      if (installedChunkData) {
        // 正在加载中
        promises.push(installedChunkData[2])
      } else {
        // 将 promise 进行缓存
        var promise = new Promise(function (resolve, reject) {
          installedChunkData = installedChunks[chunkId] = [resolve, reject]
        })
        promises.push((installedChunkData[2] = promise))

        // 创建标签，用于加载 chunk 
        var script = document.createElement('script')
        var onScriptComplete

        script.charset = 'utf-8'
        script.timeout = 120
        if (__webpack_require__.nc) {
          script.setAttribute('nonce', __webpack_require__.nc)
        }
        // 设置 src
        script.src = jsonpScriptSrc(chunkId)

        // create error before stack unwound to get useful stacktrace later
        var error = new Error()
        onScriptComplete = function (event) {
          // avoid mem leaks in IE.
          script.onerror = script.onload = null
          clearTimeout(timeout)
          var chunk = installedChunks[chunkId]
          if (chunk !== 0) {
            if (chunk) {
              var errorType =
                event && (event.type === 'load' ? 'missing' : event.type)
              var realSrc = event && event.target && event.target.src
              error.message =
                'Loading chunk ' +
                chunkId +
                ' failed.\n(' +
                errorType +
                ': ' +
                realSrc +
                ')'
              error.name = 'ChunkLoadError'
              error.type = errorType
              error.request = realSrc
              chunk[1](error)
            }
            installedChunks[chunkId] = undefined
          }
        }
        var timeout = setTimeout(function () {
          onScriptComplete({ type: 'timeout', target: script })
        }, 120000)
        script.onerror = script.onload = onScriptComplete
        // 写入 script 标签
        document.head.appendChild(script)
      }
    }
    return Promise.all(promises)
  }

  // expose the modules object (__webpack_modules__)
  __webpack_require__.m = modules

  // expose the module cache
  __webpack_require__.c = installedModules

  // define getter function for harmony exports
  __webpack_require__.d = function (exports, name, getter) {
    if (!__webpack_require__.o(exports, name)) {
      Object.defineProperty(exports, name, {
        enumerable: true,
        get: getter
      })
    }
  }

  // define __esModule on exports
  __webpack_require__.r = function (exports) {
    if (typeof Symbol !== 'undefined' && Symbol.toStringTag) {
      Object.defineProperty(exports, Symbol.toStringTag, {
        value: 'Module'
      })
    }
    Object.defineProperty(exports, '__esModule', { value: true })
  }
  
  __webpack_require__.t = function (value, mode) {
    if (mode & 1) value = __webpack_require__(value)
    if (mode & 8) return value
    if (mode & 4 && typeof value === 'object' && value && value.__esModule)
      return value
    var ns = Object.create(null)
    __webpack_require__.r(ns)
    Object.defineProperty(ns, 'default', {
      enumerable: true,
      value: value
    })
    if (mode & 2 && typeof value != 'string')
      for (var key in value)
        __webpack_require__.d(
          ns,
          key,
          function (key) {
            return value[key]
          }.bind(null, key)
        )
    return ns
  }

  // getDefaultExport function for compatibility with non-harmony modules
  __webpack_require__.n = function (module) {
    var getter =
      module && module.__esModule
        ? function getDefault() {
            return module['default']
          }
        : function getModuleExports() {
            return module
          }
    __webpack_require__.d(getter, 'a', getter)
    return getter
  }

  // Object.prototype.hasOwnProperty.call
  __webpack_require__.o = function (object, property) {
    return Object.prototype.hasOwnProperty.call(object, property)
  }

  // __webpack_public_path__
  __webpack_require__.p = ''

  // on error function for async loading
  __webpack_require__.oe = function (err) {
    console.error(err)
    throw err
  }
	// 1 定义变量存放数组，储存懒加载模块
  var jsonpArray = (window['webpackJsonp'] = window['webpackJsonp'] || [])
  // 2 存放原生数组的 push 方法
  var oldJsonpFunction = jsonpArray.push.bind(jsonpArray)
  // 3 重写原生的 push 方法，在被懒加载中的文件中调用
  jsonpArray.push = webpackJsonpCallback
  jsonpArray = jsonpArray.slice()
  for (var i = 0; i < jsonpArray.length; i++)
    webpackJsonpCallback(jsonpArray[i])
  var parentJsonpFunction = oldJsonpFunction

  // Load entry module and return exports
  return __webpack_require__((__webpack_require__.s = './src/index.js'))
})({
  './src/index.js':
    /*! no static exports found */
    function (module, exports, __webpack_require__) {
      let oBtn = document.getElementById('btn')

      oBtn.addEventListener('click', function () {
        __webpack_require__
          .e(/*! import() | login */ 'login')
          .then(
            __webpack_require__.t.bind(
              null,
              /*! ./login.js */ './src/login.js',
              7
            )
          )
          .then((login) => {
            console.log(login)
          })
      })

      console.log('index.js执行了')
    }
})
```
