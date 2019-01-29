# taro-best-practices

使用 Taro 开发微信小程序、编译 H5 + React Native 的最佳实践

鉴于只开发微信小程序跟实现多端编译需要考虑的点有所不同，将这两部分内容进行了拆分：

* 使用 Taro 开发微信小程序的最佳实践（本页面）
* [使用 Taro 编译小程序 + H5 + React Native 的最佳实践](https://github.com/js-newbee/taro-best-practices/multi-platform/README.md)

## 设置路径别名 alias

Taro v1.2.0 版本已支持设置路径别名

``` js
// config/index.js
const path = require('path')
// ...
alias: {
  '@components': path.resolve(__dirname, '..', 'src/components'),
  '@utils': path.resolve(__dirname, '..', 'src/utils')
}
```

但若要在 Sass 中使用别名，如 `@styles` 指向 `src/styles`：

``` sass
@import "@styles/theme.scss";
```

还需要额外的配置（Taro 对样式的处理是 node-sass -> postcss，在 sass 这步就报错了，不能用 postcss-import 插件解决）：

``` js
// config/index.js
plugins: {
  sass: {
    importer: function(url) {
      const reg = /^@styles\/(.*)/
      return {
        file: reg.test(url) ? path.resolve(__dirname, '..', 'src/styles', url.match(reg)[1]) : url
      }
    }
  }
}
```

备注：目前资源引用时仍无法使用别名，如 `background: url('@assets/logo.png')`，解决中


## 通过环境变量实现 config 的多元控制

通过 `npm run dev`、`npm run build` 等命令可以区分 config，如果在某环境下还需要实现进一步区分，可以在 package.json 中的 scripts 加入环境变量

``` js
"scripts": {
  "dev:weapp:mock": "MOCK=1 npm run dev:weapp"
}
```

`MOCK=1` 可以在 config 中通过 `process.env.MOCK` 访问到
