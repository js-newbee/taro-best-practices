# 使用 Taro 编译小程序 + H5 + React Native 的最佳实践

实际上，用 Taro 要做到适配 H5 还算容易，但要适配 React Native 的难度是很大的，举一例子：[Taro UI](https://github.com/NervJS/taro-ui)（Taro 团队出的组件库）至今仍未支持 RN。但也不是说适配 RN 完全不可能，坑总是要踩的，经验是实践出来的。

## 样式管理

样式是实现多端编译的核心挑战，因为 RN 端的样式只是实现了 CSS 的一个子集，具体可参考 [React-Native 样式指南](https://github.com/doyoe/react-native-stylesheet-guide)。

因此，要做到适配 RN，首先得遵循 React Native 的约束：

1. 只用 flex 布局
2. 只用 class 选择器，不用标签、子代、伪类等选择器
3. 图片统一用 Image 标签引入，不用 background 的方式
4. 文本要用 Text 标签包裹，文本的样式要加在 Text 标签上

对于第 1 点只用 flex 布局，可以在 app.scss 中加入全局样式，让微信小程序与 RN 保持一致：

``` scss
/* RN 中 View 标签的默认样式 */
view {
  display: flex;
  flex-direction: column;
  position: relative;
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

## 支付宝小程序

### 网络请求

支付宝小程序的网络请求默认的 content-type 是 application/x-www-form-urlencoded，若将 content-type 设置为 application/json，还需要手动将 data 转为字符串，否则会出现开发者工具中网络请求正常，但体验版网络请求异常的情况

``` js
Taro.request({
  url,
  data: method === 'POST' ? JSON.stringify(data) : data,
  method,
  header: { 'Content-Type': 'application/json' }
})
```
