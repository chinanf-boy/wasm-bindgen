
# 设计`wasm-bindgen`

本节旨在深入探讨如何`wasm-bindgen`今天内部工作,专门为Rust. 如果您将来阅读这篇文章很远,它可能不再是最新的,但随时可以打开一个问题,我们可以尝试回答问题和/或更新这个问题!

## 基础: ES模块

首先要了解的事情`wasm-bindgen`它是从根本上建立在ES模块的基础上的. 换句话说,这个工具采取了自以为是的文件的立场*应该被视为ES模块*. 这意味着你可以`import`从一个wasm文件,使用它`export`来自普通JS文件的-ed功能等. 

现在不幸的是,在撰写本文时,wasm interop的界面并不是很丰富. Wasm模块只能调用函数或导出专门处理的函数`i32`,`i64`,`f32`,和`f64`. 坏消息!

这就是这个项目的用武之地是用类,JS对象,Rust结构,字符串等更丰富的类型来增强wasm模块的"ABI". 但请记住,一切都基于ES模块!`wasm-bindgen`这意味着编译器实际上正在生成一个"破坏"的各种各样的wasm文件. `wasm-bindgen`例如,rustc发出的wasm文件没有我们想要的接口. 相反,它需要工具来后处理文件,生成一个`foo.js`和`foo_bg.wasm`文件. 该`foo.js`file是用JS表示的所需接口 (类,类型,字符串等) 和`foo_bg.wasm`模块简单地用作实现细节 (它是从原始版本轻微修改的`foo.wasm`文件) . 

随着WebAssembly中的更多功能随着时间的推移而稳定 (如主机绑定) ,JS文件有望变得越来越小. 它不太可能永远消失,但是`wasm-bindgen`旨在尽可能地遵循WebAssembly规范和建议以优化JS / Rust. 

## 基础#2: 在Rust中不引人注目

在事情上更生锈的一面`wasm-bindgen`箱子的设计理想情况下尽可能减少对Rust箱子的影响. 理想情况下`#[wasm_bindgen]`属性在关键位置注释,否则您将参加比赛. 该属性致力于既不发明新语法又适用于现有的习语. 

例如,库可能会暴露普通Rust中的一个函数,如下所示: 

```rust
pub fn greet(name: &str) -> String {
    // ...
}
```

与`#[wasm_bindgen]`将它导出到JS所需要做的就是: 

```rust
#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    // ...
}
```

此外,这里的设计对Rust的干预最少,这样我们就可以轻松利用即将到来的优势[主机绑定][host]提案. 理想情况下,您只需升级`wasm-bindgen`-the-crate以及您的工具链,您可以立即获得对主机绑定的原始访问权限! (虽然这仍然有点远......) 

[host]: https://github.com/WebAssembly/host-bindings
