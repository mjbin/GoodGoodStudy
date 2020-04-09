# 一次项目升级踩错记

因为上次升级没做总结，嗯，一年多了，故这次特意升级多一次总结一下，然而5.0快出了。。。

> ## 一、webpack

1、先升级webpack4，还要安装webpack-cli

```
yarn add webpack
yarn add webpack-cli 
```

1、Webpack 4 配置, 必须配置``mode``属性，可选值有``development, production, none``，先安排上。  
2、运行一下``yarn dev``，抛错``webpack.optimize.CommonsChunkPlugin has been removed, please use config.optimization.splitChunks instead.``

![CommonsChunkPlugin](https://raw.githubusercontent.com/mjbin/good-good-study/master/img/Snipaste_2020-04-03_15-45-39.png)

是的没错，字面意思，``CommonsChunkPlugin``不要了，用``optimization``配置，具体配置参考文档，我们这里先去掉``CommonsChunkPlugin``，需要补充什么配置先暂停，因为继续``yarn``还是会报错。

``optimization.minimizer``配置我们主要用来在production模式下压缩代码，所以我们先配置上，压缩js我们用``terser-webpack-plugin``插件，配置大概如下，压缩css用``optimize-css-assets-webpack-plugin``，可以配置``cssnano``使用。其他参数可参考文档。
```js
minimizer: isProd ? [
  // 压缩js
  new TerserPlugin({
    cache: true,
    parallel: true, // 多线程
    sourceMap: isProd,
  }),
  // 压缩css
  new OptimizeCSSAssetsPlugin({
    cssProcessor: require('cssnano'), //引入cssnano配置压缩选项
    cssProcessorOptions: { 
      discardComments: { removeAll: true }
    },
    canPrint: true
  }), 
] : [],
```
然后我们还想提取一个vendor第三方文件，提取vender文件可以通过配置``optimization.splitChunks.cacheGroups``实现，通过配置``mini-css-extract-plugin``和cacheGroups我们可以提取css为一个单独文件，``mini-css-extract-plugin``需要在client端配置，然后server端和client端需要配置上rules来处理css

```js
splitChunks: {
  cacheGroups: {
    // 提取node_modules依赖的包为vendor
    vendor: {
      test: /[\\/]node_modules[\\/]/,
      name: 'vendors',
      chunks: 'all'
    },
  }
}
```

3、然后继续``yarn dev``，提示引入的vue组件都报错了，升级一下vue-loader，vue-loader引入方式有改变，

> ``const VueLoaderPlugin = require('vue-loader/lib/plugin');``  

还要在plugin中配置上。``vue-loader``和``vue-template-compiler``需要一起升级，官方文档说：
```
每个 vue 包的新版本发布时，一个相应版本的 vue-template-compiler 也会随之发布。编译器的版本必须和基本的 vue 包保持同步，这样 vue-loader 就会生成兼容运行时的代码。这意味着你每次升级项目中的 vue 包时，也应该匹配升级 vue-template-compiler。
```
所以把``vue``也升级了，``vue``的版本和``vue-template-compiler``一致。

4、继续执行继续报错，因为需要不同的loader对vue文件的各个部分(js, css等)编译，所以下面会报了css的问题

![css-error](https://raw.githubusercontent.com/mjbin/good-good-study/master/img/Snipaste_2020-04-03_16-10-30.png)


此处升级一下``css-loader``和``less-loader``。 然后module.rules 配置一下规则
```javascript
{
  test: /\.(le|c)ss$/,
  use: [
    'vue-style-loader',
    'css-loader',
    'less-loader',
  ],
}
```
基本到此，项目可以正常启动。

5、项目url-loader 从1.x 版本升级到4.0版本，发现html引入的图片不能正常显示
主要是4.0版本的配置esModule 默认值为ture导致的，但是需要引入file-loader在build的时候才不会报错
``` javascript
{
  test: /\.(png|jpg|gif)$/,
  loader: 'url-loader',
  options: {
    esModule: false, // 这里设置为false
    fallback: require.resolve('file-loader'),
  }
}
```

> ## 二、babel

1、升级babel到最新7.9版本 和 eslint

2、babel 7.4 版本后就不推荐使用@babel/polyfill，而直接包含core-js / stable（以充实ECMAScript功能）和regenerator-runtime / runtime（需要使用转译的生成器函数），所以此处我引入``core-js``和``@babel/runtime``
```
As of Babel 7.4.0, this package has been deprecated in favor of directly including core-js/stable (to polyfill ECMAScript features) and regenerator-runtime/runtime (needed to use transpiled generator functions):
```

```js
npm install --save-dev @babel/core @babel/cli @babel/preset-env
```
.babelrc仅需要配置两个参数，presets和plugins，v7版本以上已经不在推荐使用stage-0等预设编译规则，所以我们的配置设置为
```json
{
  "presets": [
    [
      "@babel/env",
      {
        "targets": {
          "edge": "17",
          "firefox": "60",
          "chrome": "67",
          "safari": "11.1"
        },
        "useBuiltIns": "usage",
        "corejs": 3
      }
    ]
  ],
  "plugins": [
    "@babel/syntax-dynamic-import",
    [
      "@babel/transform-runtime"
    ],
    "syntax-export-extensions",
    "transform-vue-jsx"
  ]
}
```
部分plugin如``babel-plugin-transform-vue-jsx``，需要依赖``@babel/plugin-syntax-jsx``，所以也需要一起装上

3、其他如vue-eslint-parser，babel-eslint, eslint-plugin-vue, eslint-plugin-html, eslint-plugin-import等可参考文档使用