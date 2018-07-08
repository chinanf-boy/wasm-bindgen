
# 自定义导入行为

该`#[wasm_bindgen]`macro支持大量配置,用于精确控制导入导入的方式以及它们在JS中映射的内容. 本节旨在有希望成为可能性的详尽参考!

-   `module`和`version`- 我们见过`module`到目前为止表明我们可以从哪里导入物品`version`也允许: 

    ```rust
    #[wasm_bindgen(module = "moment", version = "^2.0.0")]
    extern {
        type Moment;
        fn moment() -> Moment;
        #[wasm_bindgen(method)]
        fn format(this: &Moment) -> String;
    }
    ```

    该`module`key用于配置从中导入每个项目的模块. 该`version`key不会影响生成的wasm本身,而是对像这样的工具的信息指令[WASM包-]. 像wasm-pack这样的工具会生成一个`package.json`为了你和`version`列在这里的时候`module`也是一个NPM包,将对应于写下来的内容`package.json`. 

    换句话说就是用法`module`作为NPM包的名称和`version`因为版本要求允许您在Rust中内联,它依赖于NPM生态系统并从这些包中导入功能. 当捆绑一个像工具[WASM包-]一切都将自动与捆绑商联系,你应该好好去!

    请注意`version`是*需要*如果`module`不是从一开始`./`. 如果`module`以. . 开始`./`那么提供版本是错误的. 

[wasm-pack]: https://github.com/rustwasm/wasm-pack

-   `catch`- 此属性允许捕获JS异常. 这可以附加到任何导入的函数,函数必须返回一个`Result`哪里`Err`有效载荷是一个`JsValue`,像这样: 

    ```rust
    #[wasm_bindgen]
    extern {
        #[wasm_bindgen(catch)]
        fn foo() -> Result<(), JsValue>;
    }
    ```

    如果导入的函数抛出异常则`Err`将返回,但引发的异常,以及其他方式`Ok`与函数的结果一起返回. 

    默认`wasm-bindgen`当wasm调用最终抛出异常的JS函数时,将不执行任何操作. 现在,wasm规范不支持堆栈展开,因此不支持Rust代码**不会执行析构函数**. 不幸的是,这可能导致Rust中的内存泄漏,但是只要wasm实现捕获异常,我们一定会添加支持!

-   `constructor`- 这用于表示被绑定的函数应该实际转换为a`new`JS中的构造函数. 最后一个参数必须是从JS导入的类型,它将在JS中使用: 

    ```rust
    #[wasm_bindgen]
    extern {
        type Foo;
        #[wasm_bindgen(constructor)]
        fn new() -> Foo;
    }
    ```

    这将附上`new`功能`Foo`类型 (暗示`constructor`并且在JS中调用此函数时它将等效于`new Foo()`. 

-   `method`- 这是通过方法等向导入的对象添加方法或以其他方式访问对象的属性的门户. 这应该是为了做相同的表达式`foo.bar()`在JS中. 

    ```rust
    #[wasm_bindgen]
    extern {
        type Foo;
        #[wasm_bindgen(method)]
        fn work(this: &Foo);
    }
    ```

    a的第一个参数`method`注释必须是附加方法所附类型的借用引用 (不可变,共享) . 在这种情况下,我们可以像这样调用这个方法`foo.work()`在JS (其中`foo`有类型`Foo`) . 

    在JS中,这个调用将对应于访问`Foo.prototype.work`然后在调用import时调用它. 注意`method`默认情况下意味着经历`prototype`获取函数指针. 

-   `js_namespace`- 此属性指示通过特定名称空间访问JS类型. 比如说`WebAssembly.Module`API都是通过`WebAssembly`命名空间. 该`js_namespace`可以应用于任何导入,并且只要生成的JS尝试引用名称 (如类或函数名称) ,就可以通过此命名空间访问它. 

    ```rust
    #[wasm_bindgen]
    extern {
        #[wasm_bindgen(js_namespace = console)]
        fn log(s: &str);
    }
    ```

    这是如何绑定的示例`console.log(x)`在Rust. 该`log`函数将在Rust模块中可用,并将作为调用`console.log`在JS中. 

-   `getter`和`setter`- 这两个属性可以结合使用`method`表明这是一个getter或setter方法. 一个`getter`默认情况下,-tagged函数访问与getter函数同名的JS属性. 一个`setter`的功能名称目前需要以"set"开头\_"它访问的属性是"set之后的后缀\_". 

    ```rust
    #[wasm_bindgen]
    extern {
        type Foo;
        #[wasm_bindgen(method, getter)]
        fn property(this: &Foo) -> u32;
        #[wasm_bindgen(method, setter)]
        fn set_property(this: &Foo, val: u32);
    }
    ```

    我们在这里导入`Foo`键入并定义访问每个对象的能力`property`属性. 这里的第一个函数是一个getter,将在Rust中可用`foo.property()`,后者是可以访问的setter`foo.set_property(2)`. 请注意,这两个函数都有`this`他们被标记的论点`method`. 

    最后,您还可以将参数传递给`getter`和`setter`用于配置访问哪个属性的属性. 显式指定属性时,方法名称没有限制. 例如,以下内容相当于上述内容: 

    ```rust
    #[wasm_bindgen]
    extern {
        type Foo;
        #[wasm_bindgen(method, getter = property)]
        fn assorted_method_name(this: &Foo) -> u32;
        #[wasm_bindgen(method, setter = "property")]
        fn some_other_method_name(this: &Foo, val: u32);
    }
    ```

    JS中的属性可以通过访问`Object.getOwnPropertyDescriptor`. 请注意,这通常仅适用于类类定义的属性,这些属性不仅仅是任何旧对象的附加属性. 要访问对象上的任何旧属性,我们可以使用...

-   `structural`- 这是一面旗帜`method`注释表明应该以结构方式访问被访问的方法 (或具有getter / setter的属性) . 例如,方法是*不*通过访问`prototype`和属性直接在对象上访问而不是通过`Object.getOwnPropertyDescriptor`. 

    ```rust
    #[wasm_bindgen]
    extern {
        type Foo;
        #[wasm_bindgen(method, structural)]
        fn bar(this: &Foo);
        #[wasm_bindgen(method, getter, structural)]
        fn baz(this: &Foo) -> u32;
    }
    ```

    这里的类型,`Foo`,不需要存在于JS中 (它没有被引用) . 相反,wasm-bindgen将生成将访问传入的JS值的填充程序`bar`财产到或`baz`财产 (取决于职能) . 

-   `js_name = foo`- 这可以用于绑定JS中的不同函数,而不是Rust中定义的标识符. 例如,您还可以在JS中为多态函数定义多个签名: 

    ```rust
    #[wasm_bindgen]
    extern {
        type Foo;
        #[wasm_bindgen(js_namespace = console, js_name = log)]
        fn log_string(s: &str);
        #[wasm_bindgen(js_namespace = console, js_name = log)]
        fn log_u32(n: u32);
        #[wasm_bindgen(js_namespace = console, js_name = log)]
        fn log_many(a: u32, b: JsValue);
    }
    ```

    所有这些功能都会调用`console.log`在Rust中,但每个标识符在Rust中只有一个签名. 
