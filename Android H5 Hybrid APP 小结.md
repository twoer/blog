### 前端技术栈
Vue + VueRouter + Vuex + Sass + Axios + Mint UI + Better-Scroll


##### Vue-Cli

基于 vue-cli 修改的 [vue-cli](https://github.com/twoer/vue-cli-demo "vue-cli-demo") ，该配置支持外部 `sass 、 .vue`  文件 中 sass 全局 px2rem（包含 font size dpr 转换） 配合 `flexible` 使用，较完美的适配各种机型；支持全局 loader sass mixin，不用额外 import 。

##### Mint UI

基于 mint-ui 修改的 [mint-ui](https://github.com/twoer/mint-ui-rem-dpr/tree/ah-light "mint-ui") ，去除了不需要的组件，仅引入了 `button、cell、field、tab、switch、radio` 等纯 css 组件，基于 `postcss-px2rem-dpr` 做了 `rem` 适配。

##### Axios

基于 Axios 实现全局 ajax 处理，Request 数据转换、加签，Response 数据转换、全局错误处理，测试环境 mock 数据转发。

注意事项

-  报文加签前需要做 `encode`，但是 前后端的 `encode` 默认实现可能不一致，需要双方约定，否则某些特殊字符会导致加签验密失败。


##### Better-Scroll

基于 [better-scroll](https://github.com/ustbhuangyi/better-scroll/blob/master/README_zh-CN.md "better-scroll") 实现了项目内的 `收款人、银行、城市、分支行` 选择页面，提高页面大量数据时的滑动体验。


### H5 与 APP 双向通讯

使用 `window.prompt` 方式实现以 `AHRCUAPP://(functionName)(callbackId)(params)` 为通讯协议的`jsbridge` ，该 `bridge` 主要包含 环境检测、H5 与 APP 通讯、APP 主动下发事件到 H5。

注意事项

-  使用时，需要注意 callbackId 的生成方式，避免重复；

- 正常场景一次通讯，一次 callback，用完即时销毁；有些场景 一次通讯可能需要多次 callback，比如 输入密码时，每输入一次，页面需要显示一个 圆点，这时候就需要双方约定 `keep callback`；

- APP 下发事件到 H5 使用的是 订阅方式，可能会出现同一个 webview 中 多个页面 订阅同一个事件的情况，订阅标记需要增加页面标识；

- 前端需要对参数做  `encode`， APP端 需要对特定的字符做处理，比如 `%`。


### 业务场景

##### 权限控制
该项目有三种权限，游客、自助注册用户、渠道用户，进入 APP 未登录时，均为游客权限，游客可浏览一些免登录的模块，做具体业务时会需要先登录，登录后需要区分 自助注册用户 和 渠道用户，某些特定功能 自助注册用户 无操作权限，整个权限模块以 `Path` 为 `Key` 配置在后管，H5 初始化前会先拉取一次游客权限，登陆/登出后重新触发菜单拉取，在 `Vue Router beforeEach` 事件中根据 `Path` 做权限判断。

注意事项

- 当游客点击业务功能菜单时，可能该菜单权限需要为 渠道用户，但 游客登录后的 角色是 自助注册用户，针对这种情况，登录成功后，需要跳转到统一页面，进行用户角色认证再做后续操作；

- 整个 H5，有些链接是需要打开新的 Webview，有的是在当前 Webview 内跳转，这个是在 Link 上加参数标记，在 `Vue Router beforeEach` 中做处理，根据参数区分是否打开新的 Webview；

- 页面跳转，无论是打开新的 Webview 还是在当前 Webview 内调整，都需要即时更新 APP 标题，这个也是在 `Vue Router beforeEach` 中做处理，前提时需要 一套 `Path` 与 `Title` 的映射配置；


##### 多页面数据同步
比如转账页面 输入 `转账金额、收款账号` 后，再选择 `收款银行`，点击 下一步 到 输入密码页面时，再返回到上一个页面，多个页面切换后需要做数据缓存，这里是使用的 `Vuex` 作为状态管理，同一个交易内多个页面 在 `Router Meta` 中配置为同一个 `Group`，公共选择页面配置为 `Selector Group`，当页面在同一个 `Group 或 Selector Group` 内跳转时，保存数据到 `Vuex`，离开 `Group` 就清除 `Vuex`。

##### 何时 history back，何时是 close webview
现有项目中部分模块是打开新的 `Webview`，小模块在该 `Webview` 内跳转，某些页面可能有 `返回`按钮，当点击 `返回`时较难处理这个按钮是需要做 `histroy back` 还是 `clsoe webview`，这边写了公共的 `back` 组件，当执行 `back` 时，增加一个定时处理，如果 100ms 内没有执行到组件的`destroyed `，说明无法执行 `history back`，那么就调用 `close webview`，如果 `destroyed ` 执行成功就清空定时并继续执行  `history back`。



### 开发调试便捷性

开发阶段有些接口可能需要走线上，有些接口需要走本地 Mock 数据，或接口环境经常切换（特别我们这边后端接口也依赖第三方接口的情况），前端可以结合 配置 和 `Axios` 来实现动态切换测试数据源；


浏览器中无法执行 APP 提供的接口，比如 打开新 Webview、Alert、Confirm 等，可以结合 `jsbridge` 实现一套 `proxy` 代码，当调用 APP 接口检测到非 APP 环境时，调用可在浏览器运行的 `proxy` 代码，保证后续逻辑继续执行。


打包不同环境代码，可以配置 `build.js` 和 `webpack config` ，让其支持传参，`npm run build` 时传不同的参数。

### 兼容性处理

低端版本 flex 布局适配，基于 `autoprefixer` 配置 `browserslist` 自动转换。


新的 ES6 API，针对部分手机不支持的情况，可以全局引入 `babel-polyfill`，但是会导致项目增加 `80KB` 左右，谨慎使用，目前项目中没有使用新的 API，仅遇到了 `Promise` 不兼容的情况，后引入 `es6-promise` 解决，改模块大小在 `10 KB` 左右 。


### 代码分享

我自己写的 [根据拼音高亮汉字](https://twoer.github.io/search-hanzi-by-pinyin/example/ "根据拼音高亮汉字")，主要用于 `收款人、银行、城市` 搜索














