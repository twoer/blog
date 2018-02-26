### 问题概况

安卓手机 APP 服务端部署新版应用后（包含 H5 资源），部分客户端无法即时更新，需要清除缓存才能生效。

### 缓存机制

##### 常见静态资源缓存策略

**Last-Modified**：请求返回头中包含该资源的最后修改时间，当再次请求该资源时，请求头中会包含 `If-Modified-Since（上次服务端返回的最后修改时间）` ，服务端做比较后，如果最后修改一致则表示没有更新，返回 `304`，客户端继续使用本地缓存，否则返回`200`更新资源。

**ETag**：与 `Last_modified` 类似，只不过把判断依据由资源最后修改时间变成资源的 Hash 值，上送字段为 `If-None-Match`。

以上两种机制都有一个共同点，就是请求资源时，都需要往服务端发送一次请求，询问该资源是否需要更新。

**Expires **：请求头中直接返回该资源的有效时间，即在该时间内，后续请求该资源时直接走本地缓存，过了约定时间则会再次向服务端发送请求，目前大部分 Web 服务器支持多种配置方式，比较常见的是基于资源最后访问时间或资源最后修改时间，比如 Apache 的两种配置写法：   
`ExpiresByType text/html "access plus 1 hours"`

`ExpiresByType image/gif "modification plus 5 hours"`

**Cache-Control**：缓存控制，常用配置，
`max-age`：缓存时间，从请求时间 到 过期时间 之间的秒数；

`no-cache`：强制每次请求直接发送给源服务器，而不经过本地缓存版本的校验；

`no-store`：强制缓存在任何情况下都不要保留任何副本。

> 当 `Cache-Control` 和 `Expires` 同时存在时， `Cache-Control` 配置会覆盖  `Expires` 。


##### Android Webview 缓存策略
**LOAD_CACHE_ONLY**:  不使用网络，只读取本地缓存数据；
**LOAD_DEFAULT**： 根据`Cache-Control`决定是否从网络上取数据；
**LOAD_CACHE_NORMAL**： API level 17 中已经废弃, 从API level 11开始作用同LOAD_DEFAULT模式；
**LOAD_NO_CACHE**：不使用缓存，只从网络获取数据；
**LOAD_CACHE_ELSE_NETWORK**：只要本地有，无论是否过期，或者`no-cache`，都使用缓存中的数据。

### 解决方案
根据以上规则 及 手机银行客户端 Webview 默认缓存配置为 `LOAD_DEFAULT` 的情况，如果需要服务端部署后，客户端即时更新，则需要在资源请求返回头中配置 `Cache-Control`，由于我们在第一次部署应用时忽略了该问题，并没有在服务端做 `Cache-Control` 配置，因此导致了该问题。

> 另测试发现在没有配置 `Cache-Control` 的情况下，部分安卓手机 可以基于 `Last-Modified` 做资源更新。

以 Vue JS 基于 Vue-Cli 脚手架构建的项目为例，JS、CSS、Image 等资源在打包后都会带上 MD5 值，程序入口在 `index.html` 文件，我们只需保证程序入口 `index.html` 文件即时更新即可，对此我们只需要在服务端针对 `index.html` 文件配置 `Cache-Control` 为 `no-cache,no-store` 即可。


### 后续完善方案
上述方案（经抓包发现不少应用都是该策略）虽然能够解决资源无法即时更新的问题，但会发现，客户端会重复请求 `index.html` 文件，虽然该程序入口文件可能在 `gzip` 后大小只有 `1kb` 左后，但重复的请求还是会增加客户端的加载时间和服务端的网络带宽消耗。

后续我们可以使用 `Cache-Control 的 max-age` 和 `Expires`  并结合项目情况 合理的调整缓存时间，做到资源 **即时更新**、**特定时间内缓存**、**兼容设备广**。

### 配置实例
以 IHS / Apache 为 Server，Vue-Cli 编译打包 为例
1. 开启 headers 模块；
2. 增加以下配置
![](https://lexiangla.com/assets/327a3858126711e8a2aa5254004fae61)

> 注意事项
如果需要配置 `header` 的 静态资源文件在 `IHS / Apache` 服务器中，可以使用 `FilesMatch` 做过滤，如果 `IHS / Apache` 只是用来做转发，`FilesMatch` 是无法匹配到的，请使用 `LocationMatch`。


# 其实资源缓存这块要比上述写的复杂N倍，N > 100。
















