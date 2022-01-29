# Webpack 是怎么处理 ``require`` 和 ``import`` 的

最近在将一个老项目进行前后端分离的过程中，遇到了一个有趣的挑战，解决过程中，引发了自己对 webpack 中关于 require 和 import 处理的一些思考和探索，整体过程记录如下。

### 背景

首先介绍一些问题的背景：老项目基于 koa bff 层和 handlerbar 模板（具体可以参考 [Handlebars 首页](https://www.handlebarsjs.cn/)）开发，与 html 模板不同的是，handlebars 模板中书写有特定的 mustache 语法，可以方便 server 进行一些变量的提前注入。老项目大致的渲染路径为：

1. 用户访问页面路由时候，bff server 层通过 mustache 语法对 handlebars 模板进行变量注入后，转换成 html 模板返回；
2. 浏览器侧，React 通过客户端渲染，完成页面的渲染和展示。

由于 bff 层和前端模板耦合在一起，因此每次发布都需要整体的对 bff 层和前端代码同时发布，流程耗时较长。为了减少发布耗时，经过讨论后决定将项目从原来的 bff server + handlebars 模板渲染形式改成 html 普通模板前后端分离的模式——**bff server 层只提供 api 接口，渲染过程由 react + html 模板完成**。然而在改造的过程中，我们碰到了一个比较棘手的问题：**原先通过 Mustache 语法注入的全局变量如何获取？**

老项目中，一些“用户信息”和“环境信息”这样可以提前获取的变量，是直接通过 handlebars 模板在 bff server 渲染中直接注入到 window 全局变量中的，如下图所示：

![handlebars 模板示例](https://raw.githubusercontent.com/wujc16/tech-blogs/main/Frontend/images/handlebars.jpeg)

解决这一问题，我们大致的的思路是：**在 bff server 提供获取相关信息的 API 接口，客户端渲染前，使用 fetch 的方式获取信息并注入到 window 中，获取成功后再进行渲染。** 大致的伪代码如下：

```JavaScript
import React from 'react';
import ReactDom from 'react-dom';
import { apiGet } from '@utils/api';

import App from './App';

apiGet('/api/profile').then(res => {
  // 首先进行 api 获取和全局变量的注入
  const { isProd, userId } = res.data;
  window._is_prod = isProd;
  window._user_id = userId;

  // 注入成功后渲染
  ReactDom.render(<App />, document.querySelector('#root'));
});
```

经过上面的改写后，运行过程中，确实是按照我们设想的流程运行，然而控制台这个时候却提示了很多 ``undefined`` 的相关报错，经过分析我们发现如下的问题。

假如我们有一个 ``@utils/env.ts`` 文件，内部定义了 ``getEnv`` 函数来获取当前的环境，那么根据写法的不同，可能会导致不同的结果：

1. 如果代码按照如下形式书写，那么在渲染和后续的事件响应过程中调用，由于 ``window._is_prod`` 已经赋值，不会出现问题；

    ```JavaScript
    export function getEnv() {
      return window._is_prod ? 'prod' : 'dev';
    }
    ```
2. 而如果采用如下的形式定义，``const isProd = window._is_prod`` 在其他文件 ``import`` 当前文件时就已经运行，此时接口还没获取数据，从而导致后面函数通过闭包访问到的 ``isProd`` 值为 ```undefined`。

    ```JavaScript
    const isProd = window._is_prod;

    export function getEnv() {
        return isProd ? 'prod' : 'dev';
    }
    ```

可以发现，导致上面两种不同结果的本质原因是：**获取 window 中变量的时机不同所导致的。** 因此，即使将 React 的渲染时机推迟到接口获取成功之后，由于很多代码会在 ``import`` 执行时就开始获取 ``window`` 中的数据，因此没有办法避免上述 ``undefined`` 问题的出现。

为了解决这一问题，有以下两个思路：

1. 全局查找类似 ``const isProd = window._is_prod`` 这样的赋值语句，将对 ``isProd`` 的访问改成对 ``window._is_prod`` 的访问，这样在实际的运行时，从 ``window`` 对象里面取到的值也是最新值；
2. 通过某种机制，使得 ``import`` 语句在接口获取成功之后执行，从而保证 ``const isProd = window._is_prod`` 执行是，``window`` 内的变量已经更新。

第一个思路相对简单，然而存在隐患：由于项目很大，需要改动的地方很多，非常容易出现遗漏，导致出现线上 BUG。

因此考虑基于第二种思路来解决问题。在代码中，所有的模块引入都是使用``import``关键词，然而按照规范，``import``是“编译时”调用，要将组件和模块的引入阻塞到接口获取成功之后，需要实现“运行时”调用，而符合这一要求的只有符合 Commonjs 规范的``require``关键词，因此考虑使用``require``进行模块引用。

最后，我们采用了一种比较 tricky 的方式，原先的 webpack 入口文件 src/index.js 保持不变，代码为：
```js
import React from 'react';
import ReactDom from 'react-dom';

import App from './App';

ReactDom.render(<App />, document.querySelector('#root'));
```

然后新建 ``src/new-index.js`` 文件，代码：
```JavaScript
import { apiGet } from '@utils/api';

apiGet('/api/profile').then(res => {
  // 首先进行 api 获取和全局变量的注入
  const { isProd, userId } = res.data;
  window._is_prod = isProd;
  window._user_id = userId;

  require('./index');
});
```

并且在 webpack 中将 entry 文件从 ``src/index.js`` 改成 ``src/new-index.js`` 即可。

到这一步，我们遇到的问题其实就已经解决了。然而我们更深一步思考，其实会有：

``import``实际是 ES6 的语法标准，即使是支持该特性的浏览器，也必须在 ``script`` 标签中通过添加 ``type="module"``，而 ``require`` 关键词更是 node 环境下的 Commonjs 标准，那么问题来了：**Webpack 分别是怎么处理 require 和 import 关键词，使得编译后的 Js 文件既能在浏览器环境下执行，又满足了**。

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




