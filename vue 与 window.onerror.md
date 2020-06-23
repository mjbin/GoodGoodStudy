## vue与window.onerror的一些事

作为一个渣，一直以来都没有认真去处理异常。这其实不太好，正好最近有个需求，我们内部起了个美团logan的日志系统，需要前端对接，刚好趁此机会，好好磨练一下这块短板。

1、需求未动，文档先行，先看logan的文档，因为我这边处理的是vue搭建的项目，所以接入web-sdk，细节可以查看文档，就不重点描述了。

2、因为不太可能去侵入项目原有的业务逻辑，主要是工作量大，加上我们的需求是记录错误，正确的不管，所以我们就考虑可否一步到位，有个什么东西是可以捕捉到所有错误的，没错，就是`window.onerror`

那么问题来了，当我直接在vue中使用`window.onerror`，特意写了个错误来测试，竟然没能捕捉到错误，
```js
mounted() {
  window.onerror = (message, url, line, column, error) => {
    window.console.log('log---onerror::::', message, url, line, column, error);
  };
  
  const a = undefined;
  a.includes('world'); // eslint-disable-line
},
```
# 这是为啥？
这是因为`vue.config.errorHandler`，所以没错，我们要抛弃了`window.onerror`了, 改用`vue.config.errorHandler`，都是文档没看细的锅