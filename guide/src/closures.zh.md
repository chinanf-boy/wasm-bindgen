
# 闭包

该`#[wasm_bindgen]`属性支持将一些 Rust闭包 传递给JS. 你可以做的例子是: 

```rust
#[wasm_bindgen]
extern {
    fn foo(a: &Fn()); // could also be `&mut FnMut()`
}
```

这是一个 从 JS 导入`foo`函数,其中第一个参数是 `a`: *堆栈闭包*. 您可以调用此函数带有参数`a`:`&Fn()` 在正常的 html 与 JS 中获得这个函数. 然而,当·`foo`函数使用时,JS函数将失效,并且使用它将引发异常. 

另外, 闭包还支持 参数 和 返回值 的导出,例如: 

```rust
#[wasm_bindgen]
extern {
    type Foo;

    fn bar(a: &Fn(u32, String) -> Foo);
}
```

有时,不希望这些闭包的堆栈行为影响. 例如,您希望在 JS 中通过下一次事件循环中运行`setTimeout`的闭包. 为此,您希望导入的函数返回,和 JS闭包 需要有效!

要支持此用例,您可以执行以下操作: 

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern {
    fn baz(a: &Closure<Fn()>);
}
```

该`Closure`类型定义在`wasm_bindgen`箱子和代表"长寿"的闭包. JS闭包传递给了`baz`之后, `baz`返回仍然有效,在 Rust 中 JS闭包 的有效性 与 `Closure`的生命周期 有关. 一旦`Closure`被删除, 它将释放其内部内存, 并使相应的 JS函数 无效. 

像堆栈闭包一样`Closure`也支持`FnMut`: 

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern {
    fn another(a: &Closure<FnMut() -> u32>);
}
```

这时你不能[将一个 JS闭包 传递给Rust][cbjs],在有限的情况下,你只能将一个 Rust闭包 传递给JS. 

[cbjs]: https://github.com/rustwasm/wasm-bindgen/issues/103
