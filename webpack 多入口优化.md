## 目前的方式
目前单页应用多个项目基于同一套 webpack 配置，那就需要去设置各自项目的 entry，才能分开调试和打包。webpack 的 entry 是它的四大核心之一，支持也比较灵活，可以是字符串、数组、对象。我们项目选择的是对象方式。
已现有项目为例，找医生、积分商城、生长曲线都属于独立的单页应用，他们基于同一套 webpack 配置，对应的项目结构为：
```
|—— vue-projects
		|———— webpack
		|———— findDoctor
			...
			|———— index.html
			|———— main.js
		|———— scoreMall
			...
			|———— index.html
			|———— main.js
		|———— growthCurve
			...
			|———— index.html
			|———— main.js
```
这就是大致的项目结构。webpack 是按入口进行打包，所以每个项目应该有各自的入口文件。如果有使用过脚手架 vue-cli 去生成项目应该会知道每个项目根目录下面都会有一个 main.js 作为入口。那配置 webpack entry 可以配置为:
```javascript
entry: {
	findDoctor: '../findDoctor/main.js',
	scoreMall: '../scoreMall/main.js',
	growthCurve: '../growthCurve/main.js'
}
```
这是入口的配置，在本地开发调试时，我们调试哪个项目就需要访问到哪个项目的 index.html 文件。但默认的 webpack 配置会将所有的 js 插入同一个 index.html 文件。
```
// index.html
<head>
...
</head>
<body>
  <div id="app"></div>
  <script src="/findDortor.js"></script>
  <script src="/scoreMall.js"></script>
  <script src="/growthCurve.js"></script>
</body>
```
这样会导致没法调试。所以实际我们需要的是，不同的项目能有各自的 index.html，而且各自的 html 只包含自己的 js 文件。
那可以再加一层 server 用来按路径去返回各自的 index.html。这个 index.html 本身就存放于各自项目的根目录下面。这层 server 也是基于 express 框架来做的。
```javascript
const routers = [{
  path: '/gc*', // 生长曲线
  file: 'growthCurve/index'
},{
  path: '/sm', // 积分商城
  file: 'scoreMall/index'
}{
  path: '/fd*', // 找医生
  file: 'findDoctor/index'
}];

routers.forEach(item => {
  const filePath = `${path.dirname(__dirname)}/projects/${item.file}`;
  console.log(filePath);
  app.get(item.path, (req, res) => {
    res.sendFile(`${filePath}.html`);
  });
});
``` 
那通过这样的配置，我们调试时就可以 通过访问不同的 html 分别调试不同的项目。
```
localhost:3006/fd ——> 找医生
localhost:3006/sm ——> 积分商城
localhost:3006/gc ——> 生长曲线
```
## 存在的问题
以前这种方式其实最大的问题就是 扩展性差，不够灵活，每次新建项目都需要去配置入口，配置访问 index.html 的路由。而且在路由的名称也没有太大的规范，一般都是利用项目名称简写。比如 **/fd** 代表了找医生，积分商城是 **/sm** 😂😂😂。这种可能自己人熟悉了之后好理解，如果一个新人看到可能会困惑。
## 优化方案
- 入口优化

既然 entry 需要拿到 入口名称作为 key，入口文件路径作为 value 的对象，那完全可以用 node.js 脚本去遍历项目来生成这样一个对象。默认情况下直接将项目名称当做 key，这样生成的 js 名称与项目名称一致。
``` javascript
//entry.config.js
const path = require('path');
const fs = require('fs');
const entry_files = {};

function eachFile(dir) {
  fs.readdirSync(dir).forEach(function (file) {
    const file_path = dir + '/' + file;
	const stat = fs.statSync(file_path);

    if (!stat.isDirectory()) {
      return;
    };

    const configFile = `${file_path}/entry.json`;
    if (fs.existsSync(configFile)) {
      const entryObj = JSON.parse(fs.readFileSync(configFile));

      for (let key in entryObj) {
        entry_files[key] = `${file_path}/${entryObj[key]}`
      }
      return;
    }
    
    const dirName = file;
    fs.readdirSync(file_path).forEach((file) => {
      if (path.extname(file) === '.js') {
        let baseName = path.basename(file, '.js');
        if (baseName === 'main' || baseName === 'index') {
          baseName = dirName;
        }
        entry_files[baseName] = `${file_path}/${file}`;
      }
    })
  })

  return entry_files;
}

module.exports = eachFile('./projects');
```
这段脚本会遍历 projects 下面所以项目，根据项目名生成 key，value。同时也兼容了历史问题，之前有些发布上线的项目 js 名称与项目名称不一致，如 `scoreMall` 项目线上已经发布的 js 名称为 `score_mall.js`，如果按照脚本生成的 js，那还需要重新上传新的 js，服务端也得改。所以这段脚本有支持根据在项目根目录下配置 `entry.json` 来自定义 key，value。如果存在 entry.json 文件会优先以它的配置为准。
```
// scoreMall entry.json
{
	"score_mall": 'main.js'
}
```
同时实际项目中，还有一种复杂情况就是一个项目可能也会有多个入口文件，比如当初的预约分为了 booking.js、schedule.js，而且这两个文件是独立的。所以当项目根目录下碰到多个 入口 js 时，可以直接根据文件名生成key，value。
经过这样一套改造之后 entry 的配置变成了：
```
const entry_files = require('../config/entry');
{
	...
	entry: entry_files
}
```
之后不管怎么添加项目都不再需要手动更改 entry。
- 模板读取优化

入口解决了还要再优化模板读取，只有读到了正确的模板才能访问到正确的页面。本来 webpack 提供了插件 webpackHtmlPlugin 来解决模板读取问题，大概就是有多少个独立项目就生成多少个模板。所以也是用脚本去遍历项目来生成模板数组。
其实现的效果为：
```
localhost:3006/findDoctor.html // 找医生首页
localhost:3006/scoreMall.html // 积分商城首页
```
所以是按准确的名称去得到准确的 html，表面上没问题，跳转也 ok，当子页面刷新时会 404
```
localhost:3006/findDoctor.html/doctor // 找医生列表页
```
以这个路径去刷新肯定没办法请求到 findDoctor.html。尝试过去像 express 那样配置路由时带上`*` 号解决这个问题，但发现无从配置。webpack 这个东西折腾起来还真的很搞人，似乎按文档来总是不能解决实际问题。
最终还是将这块的处理用老的方式来解决，但老的方式也可以用脚本优化。思路也都是一样，去遍历项目，最终生成路径和 html 的映射。
```javascript
[
  { path: '/cspcDoctor*', file: 'cspcDoctor/index.html' },
  { path: '/findDoctor*', file: 'findDoctor/index.html' },
  { path: '/scoreMall*', file: 'scoreMall/index.html' }
  ...
]
```
经过这样的修改之后的新项目不用再配置，访问路径也明确了。

通过优化入口和模板读取，项目的扩展性会比之前好，虽然这样的优化方案可能不是最佳的，但优化这种事情显然不是一蹴而就的，需要一步一步来。