
<meta charset="utf-8"/>

# `wasm-bindgen` [![translate-svg]][translate-list]

[translate-svg]: http://llever.com/translate.svg
[translate-list]: https://github.com/chinanf-boy/chinese-translate-list

**促进wasm模块和JavaScript之间的高级交互.**

[rustwasm/wasm-bindgen/guide 原文 commit ⏰  2018 7.7 12:](https://github.com/rustwasm/wasm-bindgen/tree/175319c1e0438e485d991b7f01abbb078797869e)

[👋 更多中文翻译](https://github.com/chinanf-boy/chinese-translate-list)

> - 使用 [translate-mds](https://github.com/chinanf-boy/translate-mds) 完成✅翻译初稿

## 生活

[If help, **buy** me coffee —— 营养跟不上了，给我来瓶营养快线吧! 💰](https://github.com/chinanf-boy/live-need-money)

# 校对🀄️

- ⏰ 开始 2018 7.4 12:00

> 本项目的 文档在 `guide`文件夹 中 的 `*.zh.md`

- [x] [0. 博客文章: "从JavaScript变rust再回来: `wasm-bindgen`故事"][post]

- [x] [1. 介绍](./guide/src/introduction.zh.md)

* * *

- [x] [2. 基本用法](./guide/src/basic-usage.zh.md)
- [x] [3. 刚刚发生了什么?](./guide/src/what-just-happened.zh.md)
- [x] [4. 我们还能做什么?](./guide/src/what-else-can-we-do.zh.md)
- [x] [5. 闭包](./guide/src/closures.zh.md)
- [x] [6. 功能 参考](./guide/src/feature-reference.zh.md)
- [x] [7. CLI 参考](./guide/src/cli-reference.zh.md)
- [x] [20. 类型 参考](./guide/src/reference.zh.md)

* * *

- [x] [8. 贡献](./guide/src/contributing.zh.md)
    - [x] [9. 内部设计](./guide/src/design.zh.md)
        - [ ] [10. Rust中的JS对象](./guide/src/design/js-objects-in-rust.zh.md)
        - [ ] [11. 将函数导出到JS](./guide/src/design/exporting-rust.zh.md)
        - [ ] [12. 将 struct 导出到JS](./guide/src/design/exporting-rust-struct.zh.md)
        - [ ] [13. 定制 exports ](./guide/src/design/export-customization.zh.md)
        - [ ] [14. 从JS导入函数](./guide/src/design/importing-js.zh.md)
        - [ ] [15. 从JS导入一个类](./guide/src/design/importing-js-struct.zh.md)
        - [ ] [16. 定制 imports](./guide/src/design/import-customization.zh.md)
        - [ ] [17. Rust类型转换](./guide/src/design/rust-type-conversions.zh.md)
        - [ ] [18. 在`wasm-bindgen`中的类型](./guide/src/design/describe.zh.md)
    - [x] [19. 团队](./guide/src/team.zh.md)



欢迎 `Issue` 和 `Pull` ❤️, 最好是 `Pull` 👏

---

[介绍博客文章: "从JavaScript变rust再回来: `wasm-bindgen`故事"][post]

[host]: https://github.com/WebAssembly/host-bindings

[post]: ./javascript-to-rust-and-back-again-a-wasm-bindgen-tale.zh.md

[![Build Status](https://travis-ci.org/rustwasm/wasm-bindgen.svg?branch=master)](https://travis-ci.org/rustwasm/wasm-bindgen)
[![Build status](https://ci.appveyor.com/api/projects/status/559c0lj5oh271u4c?svg=true)](https://ci.appveyor.com/project/alexcrichton/wasm-bindgen)
[![](http://meritbadge.herokuapp.com/wasm-bindgen)](https://crates.io/crates/wasm-bindgen)
[![](https://img.shields.io/crates/d/wasm-bindgen.svg)](https://crates.io/crates/wasm-bindgen)
[![API Documentation on docs.rs](https://docs.rs/wasm-bindgen/badge.svg)](https://docs.rs/wasm-bindgen)

将JavaScript内容导入Rust, 并将Rust内容导出为JavaScript. 

`src/lib.rs`: 

```rust
#![feature(proc_macro, wasm_custom_section, wasm_import_module)]

extern crate wasm_bindgen;
use wasm_bindgen::prelude::*;

//从浏览器 Web 导入 the `window.alert` 函数 
#[wasm_bindgen]
extern {
    fn alert(s: &str);
}

// 从 Rust 的 `greet` 函数 导出到 js, 这会变成 alert 一个 Hello messages
#[wasm_bindgen]
pub fn greet(name: &str) {
    alert(&format!("Hello, {}!", name));
}
```

使用 Rust 导出的JavaScript!

`index.js`: 

```js
// 异步 加载, 编译, 和 导入 Rust's WebAssembly
// 还有 JavaScript 接口.
import("./hello_world").then(module => {
  // Alert "Hello, World!"
  module.greet("World!");
});
```

## 指南

[📚阅读`wasm-bindgen`在这里指导!📚](https://rustwasm.github.io/wasm-bindgen)

## 执照

该项目是根据任何一个许可的

-   Apache License,Version 2.0, ([许可证APACHE](LICENSE-APACHE)要么<http://www.apache.org/licenses/LICENSE-2.0>) 
-   MIT许可证 ([LICENSE-MIT](LICENSE-MIT)要么<http://opensource.org/licenses/MIT>) 

根据你的选择. 

## 贡献wasm-bindgen

[有关 hack in 的信息,请参阅指南的"贡献"部分`wasm-bindgen`!][contributing]

除非您明确说明,否则您按照`Apache-2.0许可证`的规定有意提交包含在本项目中的任何贡献

应按上述方式进行双重许可,

不附加任何其他条款或条件. 

[contributing]: https://rustwasm.github.io/wasm-bindgen/contributing.html
