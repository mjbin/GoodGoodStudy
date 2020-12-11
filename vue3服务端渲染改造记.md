## vue3服务端渲染改造记

vue3正式发布有一段时间了，所以我们这次来看看怎么改造一下vue3的服务端渲染，先上[demo](https://github.com/mjbin/vue3-vite-ssr-demo)

首先我们先建立一个简单的demo项目
```javascript
yarn create vite-app vite-demo
```
默认项目是没有引入`vue-router`的，我们可以先安排上

然后我们开始引入[vite-ssr](https://github.com/frandiox/vite-ssr)，根据文档可以很方便改造好我们vue为服务端渲染
```javascript
yarn add vite-ssr --dev
```

### 三步走走起

1. 新建vite.config.js，然后把vite-ssr/plugin.js 引入
```js
const viteSSRPlugin = require('vite-ssr/plugin')

module.exports = {
  plugins: [viteSSRPlugin],
}
```

2. 修改main.js
```js
import viteSSR from 'vite-ssr'
import App from './App.vue'
import routes from './routes'
import './index.css'

export default viteSSR(
  App,
  { routes },
  ({ app, router, isClient, request, initialState }) => {
    // The 'request' is the original server request (undefined in browser).
    // The 'initialState' is only available in the browser and can be used to
    // pass it to Vuex, for example, if you prefer to rely on stores rather than Page props.
    router.beforeEach(async (to, from, next) => {
      next()
    })
  }
)
```

3. 调整package.json，build前可以通过命令删除旧的dist目录
```json
"scripts": {
  "dev": "vite --open",
  "prebuild": "rm -rf dist",
  "build": "vite-ssr build",
}
```

这样，基本就可以了，过程还是比较简单

build之后会生成dist/client和dist/ssr 两个文件目录，
一个服务端用，一个客户端用。

试一下启动服务，创建一个server.js，我这里可以通过express来启动一个服务
```js
const path = require('path')
const express = require('express')

const { ssr } = require('./dist/ssr/package.json')
const { default: handler } = require('./dist/ssr/main')

const server = express()

for (const asset of ssr.assets || []) {
  server.use(
    '/' + asset,
    express.static(path.join(__dirname, './dist/client/' + asset))
  )
}
// 再代理到根路径，解决图片问题
server.use('/', express.static(path.join(__dirname, './dist/client/_assets')))

server.get('*', async (req, res) => {
  const url = req.protocol + '://' + req.get('host') + req.originalUrl
  const { html } = await handler({ request: { ...req, url } })
  res.end(html)
})

const port = 8080
console.log(`Server started: http://localhost:${port}`)
server.listen(port)
```
更新package.json
```json
"scripts": {
  "dev": "vite --open",
  "prebuild": "rm -rf dist",
  "build": "vite-ssr build",
  "start": "node server"
}
```
这样`yarn start` 就可以启动构建后的web项目了。

### 问题
这次改造遇到两个问题

1. `yarn start`后，发现引入项目的图片不显示，因为图片的路径是`/logo.3b714202.png`, 而生成的其他资源如js，css的路径是`/_assets/index.161635d4.js`, 我们执行`node server`的时候只把资源挂载到`/_assets`目录下，就是下面的代码，所以这里偷懒处理，把资源也挂载根路径下，后面需要看下有没有配置可以修改图片引入的路径，这样图片就可以正常显示了
```js
for (const asset of ssr.assets || []) {
  server.use(
    '/' + asset,
    express.static(path.join(__dirname, './dist/client/' + asset))
  )
}
// 再代理到根路径，解决图片问题
server.use('/', express.static(path.join(__dirname, './dist/client/_assets')))
```

2. 改造好ssr后，我们重新`yarn dev`，竟然报错了，
```js
[vite] Optimizable dependencies detected:
express, vue

[vite] Dep optimization failed with error:
Could not load events (imported by node_modules\express\lib\express.js): ENOENT: no such file or directory, open 'E:\myGithub\vite-ssr-demo\events'
[Error: Could not load events (imported by node_modules\express\lib\express.js): ENOENT: no such file or directory, open 'E:\myGithub\vite-ssr-demo\events'] {
```
而且还是看起来毫无关系的express报错了，只能走vite的源码看看哪里报错了，最终在`node_modules/vite/dist/node/optimizer/index.js`176行找到如下提示
```js
console.log(chalk_1.default.yellow(`Tip:\nMake sure your "dependencies" only include packages that you\n` +
`intend to use in the browser. If it's a Node.js package, it\n` +
`should be in "devDependencies".\n\n` +
`If you do intend to use this dependency in the browser and the\n` +
`dependency does not actually use these Node built-ins in the\n` +
`browser, you can add the dependency (not the built-in) to the\n` +
`"optimizeDeps.allowNodeBuiltins" option in vite.config.js.\n\n` +
`If that results in a runtime error, then unfortunately the\n` +
`package is not distributed in a web-friendly format. You should\n` +
`open an issue in its repo, or look for a modern alternative.`)
```
简单来说，vite希望在安装再dependencies的依赖包是只能在浏览器环境执行的，很明显我们装了express框架是不推荐的，但是有得救，可以在vite.config.js中配置exclude，下面是获取依赖的代码，那配置exclude就可以解决我们的报错
```js
const qualifiedDeps = deps.filter((id) => {
  if (include && include.includes(id)) {
      // already force included
      return false;
  }
  if (exclude && exclude.includes(id)) {
      debug(`skipping ${id} (excluded)`);
      return false;
  }
  ...
```
最终改vite.config.js
```js
// https://github.com/vitejs/vite/blob/master/src/node/config.ts

const viteSSRPlugin = require('vite-ssr/plugin')

module.exports = {
  plugins: [viteSSRPlugin],
  optimizeDeps: {
    exclude: ['express']
  }
}
```

## 尾声
后面考虑引入vuex和ts，然后就应该完美了吧。