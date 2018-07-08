
# 功能参考

此章节将尝试说明, `wasm-bindgen`项目中实现的各种功能. 
下面这部分不是详尽无遗的,但[tests]用例也能成为你一个寻找功能例子的好地方. 

[tests]: https://github.com/rustwasm/wasm-bindgen/tree/master/tests

该`#[wasm_bindgen]`属性可以附加到函数,结构,impls 和外部模块. 

其中:

- Impls 只能包含函数,并且该属性不能附加到 impl 块中的函数 或 外部模块中的函数. 
- 任何这些类型都不允许使用生命周期参数或类型参数. 
- 其他语言模块必须有`"C"`abi (或没有) 列表 . 
- 带有`#[wasm_bindgen]`的免费函数可能有或没有 `"C"`abi 的列表,
都不必要用`#[no_mangle]`属性. 

所有 函数参数引用的结构都应该在宏本身中定义. 参数允许实现`WasmBoundary`特征,
例子是: 

-   整数-Int (u64 / i64需要`BigInt`支持) 
-   浮点数-Floats
-   借用的字符串`&str`) 
-   拥有的字符串 (`String`) 
-   export 结构 (`Foo`,注释`#[wasm_bindgen]`) 
-   export 类似 C 的枚举 (`Foo`,注释`#[wasm_bindgen]`) 
-   导入 带有注释的外部模块的类型 要`#[wasm_bindgen]`
-   借用 export 结构 (`&Foo`要么`&mut Bar`) 
-   该`JsValue`类型和`&JsValue` (不是可变引用) 
-   Vectors 和 支持的整数类型的切片 和`JsValue`类型. 

上述所有内容还可以返回,除了借用的引用
`Vec<JsValue>`目前不支持作为函数的参数. 

字符串是实现,将数据复制 in/out Rust 堆的垫片函数. 
也就是说,从 JS 传递给 Rust 的字符串 被复制到Rust堆 
(使用 生成的补丁 malloc 一些空间) ,
然后将被适当地释放. 

拥有的值通过 框 实现. 当你返回一个`Foo`, 它实际上变成了`Box<RefCell<Foo>>`,并作为指针返回给 JS . 

指针是有一个定义的 ABI ,和`RefCell`是为了确保 JS 中的 重导入 和 重命名 的安全性.
一般来说正常使用.,你不应该看到`RefCell`恐慌

JS-values-in-Rust 是 通过 索引 实现的,而索引一个生成表格是实现 JS绑定 的一部分.
此表格是通过 Rust 中指定的所有权, 以及 我们返回的绑定 进行管理. 
有关这方面的更多信息,请参阅[design doc](design.zh.html). 

目前, 所有这些构造在 JS方面 创建相对简单的代码,
大多数在 Rust与JS 中具有1: 1的匹配. 
