
# 自定义导入行为

该`#[wasm_bindgen]`macro支持大量配置,用于精确控制导出的导出方式以及它们在JS中生成的内容. 本节旨在有希望成为可能性的详尽参考!

-   `readonly`- 当附加到`pub`struct field这表明它只是来自JS,并且不会生成setter. 

    ```rust
    #[wasm_bindgen]
    pub struct Foo {
        pub first: u32,
        #[wasm_bindgen(readonly)]
        pub second: u32,
    }
    ```

    在这里`first`字段将是JS可读写的,但是`second`领域将是一个`readonly`在JS中没有实现setter并且尝试设置它的字段将引发异常. 

-   `constructor`- 当附加到Rust"构造函数"时,它将使生成的JS绑定可调用为`new Foo()`, 例如: 

    ```rust
    #[wasm_bindgen]
    pub struct Foo {
        contents: u32,
    }

    #[wasm_bindgen]
    impl Foo {
        #[wasm_bindgen(constructor)]
        pub fn new() -> Foo {
            Foo { contents: 0 }
        }

        pub fn get_contents(&self) -> u32 {
            self.contents
        }
    }
    ```

    这可以在JS中用作: 

    ```js
    import { Foo } from './my_module';

    const f = new Foo();
    console.log(f.get_contents());
    ```
