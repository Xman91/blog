Webpack前后端分离体验
=============

## 前端开发遇到的梗

在管家拉量活动中，前端作为实现的最后一环，留给我们的开发时间其实并不多。为了快速开发，常常与后台同学同步进行，因此容易出现等待接口再开发的情况，那么从少加点班，同时提高开发质量的角度出发，还是非常有必要做到前后端分离的。前后端分离的实现工具有很多，比如gulp，grunt等经典的构建方案。而webpack作为新生力量，`npm+webpack`的模式基本可以取代gulp和grunt，因此抽空进行了尝试。
在具体的实现过程中也遇到了不少坑，做一下总结。

## 什么是webpack？
官网给webpack的定义不过是
```markdown
module bundler
```
模块打包器啊，好low。作为模块打包器，webpack把所有的资源作为模块进行加载，配合主流commonJS的语法，写起来还是很容易上手的。

## 开始
#### 安装webpack
选择全局安装，公司内部`tnpm`镜像速度还是非常快的，以下全部用tnpm代替npm
```bash
tnpm install -g webpack
```
webpack的配置文件命名为`webpack.config.js`，详细的配置网上有好多，包括中文教程，当然最好还是看[官方文档](http://webpack.github.io/docs/)，这里不进行赘述，主要讲配置过程中遇到的问题：

#### 坑1：demo版css无法渲染（偶现）
首先对webpack.config.js进行如下配置：
```javascript
// webpack.config.js
var webpack = require("webpack");
module.exports = {
  entry: "./entry.js",
  output: {
      path: __dirname,
      filename: "bundle.js"
  },
  module: {
      loaders: [
          {test: /\.css$/, loader: "style!css"}
      ]
  }
};
```
其中`entry.js`作为入口文件引入`style.css`文件：
```javascript
// entry.js
var style = require('./style.css');
```
```css
/*style.css*/
body{
  background-color: yellow;
}
span{
  color: red;
}
```
就是这么简单，然而在运行过程中发现样式表无法渲染到页面，明明按照官网的步骤进行配置，于是去翻看打包文件`bundle.js`，找到对应的css部分，在样式的前面竟然出现了无数`/u0000/u0000/u0000`，这是`null`无数乱码呀--   去网上搜了一圈也没有类似的案例，且配置是按照标准来的，只好试着去修改`style.css`内的样式表，当我在`body`前面空一行或者加一个空样式`{}`时，乱码竟然消失，正常渲染！怀疑是webpack内部编译的问题，有遇到的同学可以注意下。

**之后的demo配置比较顺利，为了迎合业务需求，将demo从Mac迁移到windows开发机，并根据业务进行配置，win平台上就有比较多需要注意的地方了**

#### 坑2：8080端口与中间件
对于前后端分离，我将`webpack-dev-server`作为静态资源的服务器，`webpack-dev-server`是一个轻量的node.js express服务，需要单独安装：
```bash
tnpm install webpack-dev-server --save-dev
```
`webpack-dev-server`有两个特点:
 * 支持`--inline`的监听刷新（类似liverload），监听到文件改变就刷新页面
 * 支持`Hot Module Replacement`即模块热替换，这个功能非常棒！前端代码变动的时候无需刷新整个页面，只替换掉变化部分，毫无违和感
官方给出了配置方式，具体[需要三部](http://webpack.github.io/docs/webpack-dev-server.html#hot-module-replacement-with-node-js-api)，按照步骤配置，[此处有坑](#outputPathid)。

同时借助koa另起一个服务作为后端响应服务器，由于业务采用的是jsonp的请求方法，同时借助`koa-safe-jsonp`一个npm模块对callback进行处理，server文件的后端部分代码如下：
```javascript
// server.js
'use strict';
var koa = require('koa');
var jsonp = require('koa-safe-jsonp');

jsonp(app);
app.use(function *(){
  var json_dir = './cgiMock'+ queryString(this.request.url);
  var text = JSON.parse(fs.readFileSync(json_dir,'utf8'));
  this.jsonp =text;
});
app.listen(8080, 'localhost', function() {
  console.log('listen to 127.0.0.1:8080');
});

function queryString(url) {
  var _url = url.slice(0,url.indexOf('?'));
	if (_url!=null) return _url; return null;
}
```
可以看出思路是用ajax请求的url去拼接json文件的相对路径，然后读取文件并返回给res的body（`koa-safe-jsonp`代码中将text赋值给响应的body），作为请求端口我“理所应当”的使用了8080端口。然后运行server
```bash
node server.js
```
奇怪的事情发生了
```bash
Error: Cannot find module './cgiMock/socket.io'
```
按照提示以为是socket.io模块没有安装，赶紧`tnpm install socket.io --save-dev`安装一个，然并软--

再次运行`node server.js`依然提错误，更奇怪的是浏览器console提示跨域错误：
```markdown
XMLHttpRequest cannot load http://localhost:8080/socket.io/?EIO=3&transport=polling&t=LIYp-6b.
No 'Access-Control-Allow-Origin' header is present on the requested resource.
Origin 'http://my.pcmgr.qq.com' is therefore not allowed access.
The response had HTTP status code 500.
```
此时一头雾水（mac中明明没有这种情况），同是本地host下竟然报跨域错误。上git搜到同样的issue还不少：
```markdown
Same bug with socket.io 1.2.1 I downgrade to 0.9.17 and it works（版本降级到0.9.17，可以）
```
```markdown
I run into this issue when include client socket.io.js from a different domain than the
server... using socket.io@1.0.4（1.0.4也出现了这个问题）
```
```markdown
You should add var socket = io.connect('http://localhost:3000'); to your code..
unless you set ur Node Server with port 80.（端口改为80）
```
各种各样的回答都来了，版本降级试了下还是一样报错，而尝试更改到80端口时，确实成功了。这个让我更疑惑了：8080端口没有占用，同一个host下，截然不同的结果，试着改81、8081都正常，为了一探究竟，去wiki上搜了8080端口的具体定义：
```markdown
8080/tcp	HTTP替代端口 （http_alt）-commonly used for web proxy and caching server,
or for running a web server as a non-root user
```
**http代替端口**，好像并没有什么线索，无意中翻到了一个Stack Overflow的回答，恍然大悟:
```markdown
It looks like Socket.IO can't intercept requests starting with /socket.io/. This is because
in your case the listener is app -- an Express handler. You have to make http be listener,
so that Socket.IO will have access to request handling.

Try to replace

app.set( "ipaddr", "127.0.0.1" );
app.set( "port", 8080 );
with

http.listen(8080, "127.0.0.1");
See docs for details: http://socket.io/docs/#using-with-express-3/4
```
原来由于`Socket.IO`自身的原因，像`Express`和`koa`这些中间件无法对8080这个http代替端口进行监听，需要由'http模块'发起，至此终于顺利做到前后端的分离。


#### <span id="outputPathid">坑3：publicPath</span>
前面提到webpack强大的热替换功能，然而在实现过程需要特别注意**publicPath**项的配置，在初看文档的时候，我并没有仔细了解每个配置的含义，直接看了exampel，直觉上认为`publicPath`就是资源的路径，其实需要特别注意**publicPath**和**path**的区别，`path`为文件输出的路径，而开启`hot`模式后，实时更新的文件并不输出到`path`目录下，而是输出到`publicPath`目录的内存地址中，也就说说如果`index.html`文件下：
```html
<!-- index.html -->
<script src="../js/bundle.js"></script>
```
`script`标签的引用路径和`publicPath`的路径不一致，就算刷新页面，`index.html`还是拉取的旧文件，怎么去更新呢，因此开发环境中，建议将`publicPath`和`path`配置为一致。
```javascript
// webpack.config.js
module.exports = {
  ...
  output: {
      path:  __dirname+'/js/',
      publicPath:'/js/',
      filename: "bundle.js"
  }
  ...
};
```
这个时候实时更新文件输出到`http://localhost/js/bundle.js`下（publicPath就是配置这个路径的），而非实际的存储地址`./js/bundle.js`，需要特别注意！

## 最后
webpack一点简单的配置+npm包即可完成前后端的分离和实时刷新，这么一点工作量却可以大大提高前端的开发效率，当然用webpack完成这些工作简直是杀鸡用牛刀，在初步体验后，其实还有很多可以完善，比如：
* 生产环境和开发环境的分离
* 计算文件hash值解决缓存问题
* 与自动化测试结合
* react的配合

实现同样的功能，webpack相比于gulp可以减少不少的代码量，或多或少代表了未来的趋势，但是！作为一个已经放弃ie7/8的工具，它与业务的匹配度并不高。

因此，把webapck当做开发的小工具，用于学习ES6/Typescript等，另外在移动端H5的开发上或许可以大显身手。
