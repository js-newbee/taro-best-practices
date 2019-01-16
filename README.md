# taro-best-practices

使用 Taro 开发微信小程序的最佳实践

## 设置路径别名 alias

Taro v1.2.0 版本已支持设置路径别名

``` js
const path = require('path')
// ...
alias: {
  '@components': path.resolve(__dirname, '..', 'src/components'),
  '@utils': path.resolve(__dirname, '..', 'src/utils')
}
```

但若要在 Sass 中使用别名，如：

``` sass
@import "@styles/theme.scss";
```

还需要额外的配置（Taro 对样式的处理是 node-sass -> postcss，在 sass 这步就报错了，不能用 postcss-import 插件解决）：

``` js
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
