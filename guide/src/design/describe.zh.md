
# 将类型传达给`wasm-bindgen`

在将Rust / JS类型相互转换时谈论的最后一个方面是如何实际传达这些信息. 该`#[wasm_bindgen]`宏正在运行Rust代码的语法 (未解析) 结构,然后负责生成信息`wasm-bindgen`CLI工具稍后会读取. 

为实现这一目标,采取了一种略微不同寻常的方法. 有关Rust代码结构的静态信息通过JSON (当前) 序列化到wasm可执行文件的自定义部分. 其他信息,例如类型实际上是什么,遗憾的是直到后来在编译器中才知道由于关联类型投影和typedef之类的事情. 事实证明,我们想要传达"丰富"类型`FnMut(String, Foo,
&JsValue)`到了`wasm-bindgen`CLI,处理这一切非常棘手!

要解决这个问题了`#[wasm_bindgen]`宏生成**可执行功能**其中"描述了进口或出口的类型签名". 这些可执行函数是什么的`WasmDescribe`特质是关于: 

```rust
pub trait WasmDescribe {
    fn describe();
}
```

虽然看似简单,但这个特性实际上非常重要. 当你写,这样的导出: 

```rust
#[wasm_bindgen]
fn greet(a: &str) {
    // ...
}
```

除了我们上面谈到的垫片,JS生成宏*也*生成类似的东西: 

    #[no_mangle]
    pub extern fn __wbindgen_describe_greet() {
        <Fn(&str)>::describe();
    }

或者换句话说,它会生成调用`describe`功能. 这样做了`__wbindgen_describe_greet`shim是导入/导出的类型布局的编程描述. 然后执行这些操作`wasm-bindgen`跑!这些执行依赖于调用的导入`__wbindgen_describe`通过一个`u32`到主机,当多次调用时给出一个`Vec<u32>`有效. 这个`Vec<u32>`然后可以被重新分解成一个`enum Descriptor`它完全描述了一种类型. 

总而言之,这有点迂回,但根本不会对生成的代码或运行时产生任何影响. 所有这些描述符函数都从发出的wasm文件中删除. 
