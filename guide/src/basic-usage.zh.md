
# 基本用法

让我们实现相当于"你好,世界!"的示例

> **注意:**目前这个项目使用*最新Rust*,你可以通过 [rustup] ,并使用`rustup default nightly`

[rustup]: https://rustup.rs

如果你想要[直接进入在线示例][hello-online],但如果你想在你自己的控制台中试试,那么让我们安装我们需要的工具: 

```shell
$ rustup target add wasm32-unknown-unknown
$ cargo +nightly install wasm-bindgen-cli
```

这里的第一个命令安装了 wasm目标,这样你就可以编译 wasm,后者将安装`wasm-bindgen`, 这是我们将在稍后使用的CLI工具. 

接下来让我们来做我们的项目

```shell
$ cargo +nightly new js-hello-world --lib
```

现在,添加对这个项目的依赖`Cargo.toml`以及配置我们的构建输出: 

```toml
[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"
```

接下来我们的实际代码! 我们会写这个`src/lib.rs`: 

```rust,ignore
#![feature(proc_macro, wasm_custom_section, wasm_import_module)]

extern crate wasm_bindgen;
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern {
    fn alert(s: &str);
}

#[wasm_bindgen]
pub fn greet(name: &str) {
    alert(&format!("Hello, {}!", name));
}
```

就是这样! 如果我们写`greet`函数时,没有`#[wasm_bindgen]`属性,那样 JS 将无法用像`str`之类的类型进行通信,所以添加一个`#[wasm_bindgen]`,这样确保生成关于 函数和导入`alert` 正确的垫片

接下来让我们构建我们的项目: 

```shell
$ cargo +nightly build --target wasm32-unknown-unknown
```

在此之后你将有一个 wasm文件`target/wasm32-unknown-unknown/debug/js_hello_world.wasm`. 不要惊讶于大小,这是一个未经优化的版本!

现在我们已经生成了 wasm模块,现在是时候运行 `bindgen` 工具了! 该工具将对 rustc 生成的wasm文件 进行处理,生成 新的wasm文件 和 一组JS绑定. 让我们来调用吧!

```shell
$ wasm-bindgen target/wasm32-unknown-unknown/debug/js_hello_world.wasm \
  --out-dir .
```

见证奇迹的时刻. 该`js_hello_world.wasm`是由 rustc生成的文件, 其中包含*描述*如何通过 比目前支持更丰富的类型 进行通信. 该`wasm-bindgen`工具将解释此信息,且在 wasm文件 **更换模块** . 

之前的`js_hello_world.wasm`文件, 被解释为它是一个 ES6模块. 而后来`wasm-bindgen`生成的`js_hello_world.js`, 是应该具有 wasm文件 的预期接口,特别是 字符串,类 等丰富类型.  

`wasm-bindgen`工具还会生成, 实现此模块所需的一些其他文件. 例如: `js_hello_world_bg.wasm`是原始的wasm文件,但是经过处理的. 这是被依赖的`js_hello_world_bg.wasm`文件,就像之前一样,文件就像一个ES6模块. 

`js_hello_world.js` 调用了`js_hello_world_bg.wasm`

> 注意 **js_hello_world`_bg`.wasm** 和 **js_hello_world.wasm** 的不同

此时,您可能会将这些文件插入更大的构建系统. `wasm-bindgen`生成的文件像普通的 ES6模块 一样 (wasm文件就已经被使用了) . 截至撰写本文时,很遗憾没有很多工具本身可以做到这一点,但 Webpack的4.0测试版本 有本机的支持! 让我们来看看它,看看它是如何工作的. 

首先创建一个`index.js`文件: 

```js
const js = import("./js_hello_world");

js.then(js => {
  js.greet("World!");
});
```

请注意,我们正在使用`import(..)`这里因为Webpack[不支持][webpack-issue]同步从主块导入模块. 

[webpack-issue]: https://github.com/webpack/webpack/issues/6615

接下来我们通过创建一个JS依赖项`package.json`: 

```json
{
  "scripts": {
    "serve": "webpack-dev-server"
  },
  "devDependencies": {
    "webpack": "^4.0.1",
    "webpack-cli": "^2.0.10",
    "webpack-dev-server": "^3.1.0"
  }
}
```

和我们的 webpack 配置

```js
// webpack.config.js
const path = require('path');

module.exports = {
  entry: "./index.js",
  output: {
    path: path.resolve(__dirname, "dist"),
    filename: "index.js",
  },
  mode: "development"
};
```

我们对应的`index.html`: 

```html
<html>
  <head>
    <meta content="text/html;charset=utf-8" http-equiv="Content-Type"/>
  </head>
  <body>
    <script src='./index.js'></script>
  </body>
</html>
```

最后: 

```shell
$ npm install
$ npm run serve
```

如果你打开[本地主机: 8080](https://localhost:8080)在浏览器中你应该看到一个`Hello, world!`对话框弹出!

如果这一切都有点多,不用担心!您可以[在线执行此代码][hello-online]谢谢[WebAssembly Studio](https://webassembly.studio)或者你可以[跟随GitHub][hello-tree]查看所有必需的文件,以及设置它的脚本. 

[hello-tree]: https://github.com/rustwasm/wasm-bindgen/tree/master/examples/hello_world

[hello-readme]: https://github.com/rustwasm/wasm-bindgen/tree/master/examples/hello_world/README.md
