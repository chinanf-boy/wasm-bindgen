
# 刚刚发生了什么?

唷! 想知道刚刚发生了什么, 很多很多 - 两个大魔法: `#[wasm_bindgen]`属性和`wasm-bindgen`CLI工具. 

**该`#[wasm_bindgen]`属性**

此属性,从中 `wasm-bindgen`crate 导出,是将 Rust函数 暴露给 JS 的主文件. 这是一个程序宏 (因此需要最新的 Rust工具链) ,它将在 Rust 中生成适当的补丁程序,以便从您的 类型 转换为 JS 可以与之交互的接口. 最后,在解析后, 该属性还将一些信息序列化为`wasm-bindgen`会丢弃的输出. 

下面对属性的各个部分进行了更全面的解释,但现在使用它在 自由函数,结构,impl 块 以用于 那些结构和`extern { ... }`块. 现在还不支持一些 Rust 特性,如 泛型,生命周期 等. 

**该`wasm-bindgen`CLI工具**

下半部分,都是`wasm-bindgen`工具的内容. 这个工具打开了 rustc 生成的模块,并找到了 它的内容编码的`#[wasm_bindgen]`属性. 你可以把它想象成`#[wasm_bindgen]`属性,创建了输出模块的特殊辨识,给`wasm-bindgen`启动流程. 

这个信息给了`wasm-bindgen`, 所有它需要知道导入的JS文件. JS文件 包裹了 底层实例化的 wasm 模块 (也就是调用`WebAssembly.instantiate`) 然后为其中的 类/函数 提供包装器. 
