# 使用 Taro 编译小程序 + H5 + React Native 的最佳实践

本页内容多数整理自趣店 FED 开源的首个 Taro 多端统一实例 - 网易严选（小程序 + H5 + React Native）- <https://github.com/js-newbee/taro-yanxuan>。

## 目录

* [样式管理](#样式管理)
    * [fixed 定位的实现](#fixed-定位的实现)
* [H5](#h5)
    * [路径别名](#路径别名)
    * [跨域](#跨域)
    * [打包静态资源带 hash 值](#打包静态资源带-hash-值)
* [支付宝小程序](#支付宝小程序)
    * [网络请求](#网络请求)

## 样式管理

样式是实现多端编译的核心挑战，因为 RN 端的样式只是实现了 CSS 的一个子集，具体可参考 [React-Native 样式指南](https://github.com/doyoe/react-native-stylesheet-guide)（这个指南的版本有点旧了，内容不一定准确）。

因此，要做到适配 RN，首先得遵循 React Native 的约束：

1. 只用 flex 布局
2. 只用 class 选择器，不用标签、子代、伪类等选择器
3. 样式单位只使用 px，部分属性支持 %（[相关说明](https://github.com/facebook/react-native/commit/3f49e743bea730907066677c7cbfbb1260677d11)）
4. 图片统一用 Image 标签引入，不用 background 的方式
5. 文本要用 Text 标签包裹，文本的样式要加在 Text 标签上

对于第 1 点只用 flex 布局，需要注意 RN 的 View 标签有如下默认样式：

``` css
view {
  display: flex;
  flex-direction: column;
  position: relative;
  box-sizing: border-box;
}
```

其中 flex 的主轴与 Web 默认值不一致，因此凡是用到 `display: flex` 的地方都需要声明主轴，且需要设置部分全局样式：

``` scss
view, div {
  box-sizing: border-box;
}

button {
  /* 去掉微信小程序 button 的边框 */
  &:after {
    display: none;
  }
}
```

对于第 2 点只用 class 选择器，建议使用 BEM 管理样式：

``` scss
.comp {}
.comp__list {}
.comp__list-item {}
.comp__list-item--last {}
```

对于第 3 点，是由于 RN 不支持 background-image，需要使用 Image 标签配合 postion 实现

```js
// 建议自行封装个 ImageBackground 组件解决此类需求
<View>
  <Image style={{ position: 'absolute' }} />
</View>
```

最后，对于需要覆盖组件样式的情况，建议使用 style：

``` js
// 组件调用
<Comp textStyle={{ color: 'red' }} />

// Comp 组件
<View>
  <Text style={this.props.textStyle}>Hello</Text>
</View> 
```

之所以选用这样的方案，是基于微信小程序、RN 自身的限制及 Taro 目前支持的程度所作出的妥协，具体可参考 [Taro 在微信小程序、RN 上的样式局限](../docs/style.md) 中的详细说明，在此不展开。

### fixed 定位的实现

由于 RN 不支持 fixed 定位，有 fixed 定位的场景就需要改用 ScrollView + absolute 实现：

``` js
<View style={{ position: 'relative' }}>
  <ScrollView style={{ height: 'xxx' }}>内容区域</ScrollView>
  <View style={{ position: 'absolute' }}>固定区域</View>
</View>
```

内容区域需要用到 ScrollView，而 ScrollView 又需要设置高度，就需要去计算页面可用高度，但该值在各端上不太一致，需要根据 systemInfo 进行二次计算

``` js
// 各端返回的 windowHeight 不一定是最终可用高度（例如可能没减去 statusBar 的高度），需二次计算
const NAVIGATOR_HEIGHT = 44
const TAB_BAR_HEIGHT = 50
function getWindowHeight(showTabBar = true) {
  const info = Taro.getSystemInfoSync()
  const { windowHeight, statusBarHeight, titleBarHeight } = info
  const tabBarHeight = showTabBar ? TAB_BAR_HEIGHT : 0

  if (process.env.TARO_ENV === 'rn') {
    return windowHeight - statusBarHeight - NAVIGATOR_HEIGHT - tabBarHeight
  }

  if (process.env.TARO_ENV === 'h5') {
    return `${windowHeight - tabBarHeight}px`
  }

  return `${windowHeight}px`
}
```

## H5

### 路径别名

若要在 Sass 中使用别名，如 `@styles` 指向 `src/styles`，需要设置 h5.sassLoaderOption，具体配置见 [设置路径别名 alias](../README.md#设置路径别名-alias)。

### 跨域

跨域最好的解决方案是设置 CORS，如果没有条件，在开发时要解决跨域问题可以配置 devServer 的 proxy：

``` js
// config/dev.js

// 需要把 package.json 中 scripts 的 "dev:h5": "..." 改成：
// "dev:h5": "CLIENT_ENV=h5 npm run build:h5 -- --watch"
const isH5 = process.env.CLIENT_ENV === 'h5'
const HOST = '"http://xxx"'

module.exports = {
  defineConstants: {
    HOST: isH5 ? '"/api"' : HOST
  },
  h5: {
    devServer: {
      proxy: {
        '/api/': {
          target: JSON.parse(HOST),
          pathRewrite: {
            '^/api/': '/'
          },
          changeOrigin: true
        }
      }
    }
  }
}
```

### 打包静态资源带 hash 值

Taro 编译 H5 时静态资源是固定的文件名：

``` html
<!-- css -->
<link href="/css/app.css" rel="stylesheet"></head>

<!-- js -->
<script type="text/javascript" src="/js/app.js"></script>

<!-- 图片 -->
background:url(/static/images/bg.png)
```

但这样不利于缓存、版本控制，建议配置 webpack 给静态资源带上 hash：

``` js
// config/index.js
h5: {
  publicPath: '/',
  staticDirectory: 'static',
  output: {
    filename: 'js/[name].[hash].js',
    chunkFilename: 'js/[name].[chunkhash].js'
  },
  imageUrlLoaderOption: {
    limit: 5000,
    name: 'static/images/[name].[hash].[ext]'
  },
  miniCssExtractPluginOption: {
    filename: 'css/[name].[hash].css',
    chunkFilename: 'css/[name].[chunkhash].css'
  }
}
```

## 支付宝小程序

### 网络请求

（有待验证新版本是否已解决了该问题）支付宝小程序的网络请求默认的 content-type 是 application/x-www-form-urlencoded，若将 content-type 设置为 application/json，还需要手动将 data 转为字符串，否则会出现开发者工具中网络请求正常，但体验版网络请求异常的情况

``` js
Taro.request({
  url,
  data: method === 'POST' ? JSON.stringify(data) : data,
  method,
  header: { 'Content-Type': 'application/json' }
})
```
