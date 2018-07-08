
# `wasm-bindgen`的设计

本节旨在: 深入探讨`wasm-bindgen`如何工作, 通过 Rust语言. 如果您很久前阅读这篇文章,它可能不再是最新的,但随时可以打开一个问题,我们可以尝试回答问题和/或更新这个问题!

## 基础: ES模块

首先要了解的事情是,`wasm-bindgen` 建立在ES模块的基础上的. 换句话说,这个工具采取了 自定义的文件 *被视为ES模块*. 这意味着你可以`import`一个wasm文件 ,使用`export`得到一个普通 JS文件 的函数等. 

现在不幸的是,在撰写本文时, **wasm interop** 的接口并不是很丰富. Wasm模块只能调用函数 或 导出专门处理`i32`,`i64`,`f32`,和`f64`之类的参数. 这是个坏消息!

但这也是这个项目的用武之地, 通过使用 类,JS对象,Rust结构,字符串 等更丰富的类型来增强 wasm模块 的 "ABI". 但请记住,一切都基于 ES模块! 使用`wasm-bindgen`意味着编译器实际上正在生成一个"已经被破坏"的各种各样的wasm 文件. 例如,rustc 发出的 wasm文件 没有我们想要的接口. 相反,它需要 wasm-bindgen 处理文件,生成一个`foo.js`和`foo_bg.wasm`文件. 该`foo.js`是用 JS 表示的所需接口 (类,类型,字符串等) 和`foo_bg.wasm`模块 简单地用作实现细节 (它是从原始版本 `foo.wasm` 轻微修改的文件) . 

随着 WebAssembly 中的 更多功能 随着时间的推移而稳定 (如主机绑定) , JS文件 有望变得越来越小. 它不太可能永远消失,但是`wasm-bindgen`旨在尽可能地遵循 WebAssembly规范和建议 来优化 JS/Rust. 

## 基础#2: 在 Rust 中不引人注目

在 rsut编程方面上, `wasm-bindgen`箱子的设计理想是尽可能减少对 在 Rust 编码的影响. 理想情况下`#[wasm_bindgen]`属性在关键位置 `#注释`,不然您会看不上这个项目. 该属性致力于既不发明新语法又适用于现有的习语. 

例如,想要暴露普通 Rust 中的一个函数,如下所示: 

```rust
pub fn greet(name: &str) -> String {
    // ...
}
```

`#[wasm_bindgen]`将它 导出到JS 所需要做的就是: 

```rust
#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    // ...
}
```

此外,这里的设计对 Rust 的干预最少,这样我们就可以轻松利用即将到来的优势[主机绑定][host]提案. 理想情况下,您只需升级`wasm-bindgen`以及您的工具链,您可以立即获得对主机绑定的原始访问权限! (虽然这仍然有点远......) 

[host]: https://github.com/WebAssembly/host-bindings
