
# 从JS导入一个类

就像我们开始导出后的功能一样,我们也想导入!现在我们已经出口了`class`对于JS,我们也希望能够在Rust中导入类以调用方法等. 由于JS类通常只是JS对象,因此这里的绑定看起来与上面描述的JS对象绑定非常相似. 

像往常一样,让我们​​来看一个例子吧!

```rust
#[wasm_bindgen(module = "./bar")]
extern {
    type Bar;

    #[wasm_bindgen(constructor)]
    fn new(arg: i32) -> Bar;

    #[wasm_bindgen(js_namespace = Bar)]
    fn another_function() -> i32;

    #[wasm_bindgen(method)]
    fn get(this: &Bar) -> i32;

    #[wasm_bindgen(method)]
    fn set(this: &Bar, val: i32);

    #[wasm_bindgen(method, getter)]
    fn property(this: &Bar) -> i32;

    #[wasm_bindgen(method, setter)]
    fn set_property(this: &Bar, val: i32);
}

fn run() {
    let bar = Bar::new(Bar::another_function());
    let x = bar.get();
    bar.set(x + 3);

    bar.set_property(bar.property() + 6);
}
```

不像我们以前的进口,这个更健谈!请记住,其中一个目标`wasm-bindgen`是尽可能使用原生的Rust语法,所以这主要是为了使用`#[wasm_bindgen]`属性来解释在Rust中写下的内容. 现在这里有一些属性注释,所以让我们一个接一个地进行: 

-   `#[wasm_bindgen(module = "./bar")]`- 在导入之前看到这是声明所有后续功能都是导入表单的地方. 例如`Bar`类型将从中导入`./bar`模块. 
-   `type Bar`- 这是一个JS类的声明,作为Rust中的一个新类型. 这意味着一种新型`Bar`生成的是"不透明的",但表示为内部包含a`JsValue`. 我们稍后会看到更多. 
-   `#[wasm_bindgen(constructor)]`- 这表明绑定的名称实际上并未在JS中使用,而是转换为`new Bar()`. 此函数的返回值必须是裸类型,如`Bar`. 
-   `#[wasm_bindgen(js_namespace = Bar)]`- 此属性指示函数声明是通过命名空间命名的`Bar`JS中的类. 
-   `#[wasm_bindgen(static_method_of = SomeJsClass)]`- 这个属性类似于`js_namespace`,但不是生成一个自由函数,而是生成一个静态方法`SomeJsClass`. 
-   `#[wasm_bindgen(method)]`- 最后,此属性表示将发生方法调用. 第一个参数必须是JS结构,比如`Bar`,JS中的调用看起来像`Bar.prototype.set.call(...)`. 

考虑到所有这些,让我们来看看生成的JS. 

```js
import * as wasm from './foo_bg';

import { Bar } from './bar';

// other support functions omitted...

export function __wbg_s_Bar_new() {
  return addHeapObject(new Bar());
}

const another_function_shim = Bar.another_function;
export function __wbg_s_Bar_another_function() {
  return another_function_shim();
}

const get_shim = Bar.prototype.get;
export function __wbg_s_Bar_get(ptr) {
  return shim.call(getObject(ptr));
}

const set_shim = Bar.prototype.set;
export function __wbg_s_Bar_set(ptr, arg0) {
  set_shim.call(getObject(ptr), arg0)
}

const property_shim = Object.getOwnPropertyDescriptor(Bar.prototype, 'property').get;
export function __wbg_s_Bar_property(ptr) {
  return property_shim.call(getObject(ptr));
}

const set_property_shim = Object.getOwnPropertyDescriptor(Bar.prototype, 'property').set;
export function __wbg_s_Bar_set_property(ptr, arg0) {
  set_property_shim.call(getObject(ptr), arg0)
}
```

就像从JS导入函数一样,我们可以看到为所有相关函数生成了一堆填充程序. 该`new`静态功能有`#[wasm_bindgen(constructor)]`属性,这意味着它应该实际调用它而不是任何特定的方法`new`而是构造函数 (正如我们在这里看到的) . 静态功能`another_function`然而,被派遣为`Bar.another_function`. 

该`get`和`set`函数是方法,所以他们经历`Bar.prototype`,否则他们的第一个参数隐含地是通过加载的JS对象本身`getObject`就像我们之前看到的

一些真正的肉开始出现在Rust的一面,所以让我们来看看: 

```rust
pub struct Bar {
    obj: JsValue,
}

impl Bar {
    fn new() -> Bar {
        extern {
            fn __wbg_s_Bar_new() -> u32;
        }
        unsafe {
            let ret = __wbg_s_Bar_new();
            Bar { obj: JsValue::__from_idx(ret) }
        }
    }

    fn another_function() -> i32 {
        extern {
            fn __wbg_s_Bar_another_function() -> i32;
        }
        unsafe {
            __wbg_s_Bar_another_function()
        }
    }

    fn get(&self) -> i32 {
        extern {
            fn __wbg_s_Bar_get(ptr: u32) -> i32;
        }
        unsafe {
            let ptr = self.obj.__get_idx();
            let ret = __wbg_s_Bar_get(ptr);
            return ret
        }
    }

    fn set(&self, val: i32) {
        extern {
            fn __wbg_s_Bar_set(ptr: u32, val: i32);
        }
        unsafe {
            let ptr = self.obj.__get_idx();
            __wbg_s_Bar_set(ptr, val);
        }
    }

    fn property(&self) -> i32 {
        extern {
            fn __wbg_s_Bar_property(ptr: u32) -> i32;
        }
        unsafe {
            let ptr = self.obj.__get_idx();
            let ret = __wbg_s_Bar_property(ptr);
            return ret
        }
    }

    fn set_property(&self, val: i32) {
        extern {
            fn __wbg_s_Bar_set_property(ptr: u32, val: i32);
        }
        unsafe {
            let ptr = self.obj.__get_idx();
            __wbg_s_Bar_set_property(ptr, val);
        }
    }
}

impl WasmBoundary for Bar {
    // ...
}

impl ToRefWasmBoundary for Bar {
    // ...
}
```

在Rust,我们看到了一种新型,`Bar`,是为此类的导入生成的. 方式`Bar`内部包含一个`JsValue`作为一个例子`Bar`用于表示存储在模块的堆栈/板中的JS对象. 然后,这与我们在开始时看到JS对象的工作方式大致相同. 

打电话时`Bar::new`我们将得到一个包含在其中的索引`Bar` (这本身就是一个`u32`在内存中被剥离) . 然后,每个函数都将索引作为第一个参数传递,否则将转发Rust中的所有内容. 
