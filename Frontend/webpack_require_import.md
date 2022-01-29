# Webpack 是怎么处理 ``require`` 和 ``import`` 的

最近在将一个老项目进行前后端分离的过程中，遇到了一个有趣的挑战，解决过程中，引发了自己对 webpack 中关于 require 和 import 处理的一些思考和探索，整体过程记录如下。

### 背景

首先介绍一些问题的背景：老项目基于 koa bff 层和 handlerbar 模板。大致的渲染逻辑如下：

1. 访问 index 路由时候，koa bff server 会对 handlerbar 模板进行简单处理，并返回给前端；
2. 

由于 bff 层和前端模板耦合在一起，因此每次发布都需要整体的对 bff 层进行发布，流程耗时较长，而且不方便灰度。为了减少发布耗时，同时利用 Goofy 快捷的灰度功能，因此，决定将项目从原来的模板渲染形式改成纯 html 模板渲染的方式。但是在改造的过程中，我们碰到了如下的问题：

原项目中，用户信息和环境信息的一些变量，是直接在模板中注入到 window 全局变量中的；


在迁移 Goofy 的过程中，由于改成了普通的 HTML 模板，因此无法直接在 window 中注入全局变量，我们大致的的思路是：

1. 使用 fetch 的方式，在渲染之前完成“用户信息”和“环境信息”等的获取；
2. 获取成功后，写入到 window 全局变量中。

这样写完后，代码大概长下面这样：
```JavaScript

```

但是，在使用过程中，我们发现了，由于原来采用 handlerbar 模版的方式，拿到 html 文件的时候就已经执行了 window 全局对象的注入，所以在很多文件中（类似 ``@utils/env``）会有如下的写法：

```JavaScript
const isBoe = window.__isBoe;

export function getEnv() {
    return isBoe ? 'boe' : 'prod';
}
```

这些文件由于，直接在

为了解决这一问题，有以下两个思路：

1. 全局查找类似 ``const isBoe = window.__isBoe``  这样的赋值语句，改成下面的形式，这样在实际的运行时，从 window 对象里面取到的值也是最新值！

    ```JavaScript
    export function getEnv() {
        return window.__isBoe ? 'boe' : 'prod';
    }
    ```
2. Block import 语句的执行，在 fetch 成功并且往 window 对象里面注入

然而由于。

实际上，到这一步，我们遇到的问题其实就已经解决了。然而

## 模块

require 关键词其实是，import 则是，

实际上在，那么 webpack 是怎么处理这两类关键词，为此我们把打包模式改成 development，

``__webpack_require__`` 函数，该函数上挂了 3 个函数对象，分别为：

1. ``__webpack__require__.o``，
2. ``__webpack__require__.d``
3. ``__webpack__require__.r``

此外，比较重要的对象为：

``__webpack_modules__`` 对象，该对象是

``__webpack_module_cache__`` 对象，

### 入口文件：

1. ``__webpack_require__(${entry})`` 开始，
2. 哈哈

### ``import`` 语句

按照规范，``import`` 语句必须写在，

同样可以分成两种情况：``export`` 和 ``export default``。



1. 导出会被写成

### ``require`` 语句

``require`` 语句的处理非常简单，



替换一下顺序，

会发现，在 webpack


通过 import 语句，性能会更好一些～

下面是例子：
```JavaScript
(()=>{
  // o 是之前的 __webpack_modules__ 对象，比较有意思的是，只有使用 commonjs 
  var o={503:o=>{console.log("hhh in env"),o.exports={isBoe:function(){return!0}}}};
  var e={};
  
  // 这里是 __webpack_require__ 函数的定义
  function n(s){var t=e[s];if(void 0!==t)return t.exports;var r=e[s]={exports:{}};return o[s](r,r.exports,n),r.exports}

  (()=>{
    "use strict";
    console.log("hhhh in math");
    console.log("add(1, 2): ",3,"minus: (2, 1): ",(console.log("input is: ",2,1),1));
    const{isBoe:o}=n(503);
    console.log("isBoe: ",o(),{msg:"hello",to:"jinchao"})
  })()
})();
```

也就是说，``import`` 冠戒指




### 深入思考




