
# 将结构导出到JS

到目前为止,我们已经介绍了JS对象,导入函数和导出函数. 到目前为止,这给我们提供了相当丰富的基础,这太棒了!不过,我们有时想进一步定义JS`class`在Rust. 或者换句话说,我们希望使用从Rust到JS的方法公开一个对象,而不仅仅是导入/导出自由函数. 

该`#[wasm_bindgen]`属性可以注释a`struct`和`impl`块允许: 

```rust
#[wasm_bindgen]
pub struct Foo {
    internal: i32,
}

#[wasm_bindgen]
impl Foo {
    pub fn new(val: i32) -> Foo {
        Foo { internal: val }
    }

    pub fn get(&self) -> i32 {
        self.internal
    }

    pub fn set(&mut self, val: i32) {
        self.internal = val;
    }
}
```

这是典型的Rust`struct`具有构造函数和一些方法的类型的定义. 用结构注释结构`#[wasm_bindgen]`意味着我们将生成必要的特征impl以将此类型转换为JS边界或从JS边界转换此类型. 注释`impl`这里的块意味着内部函数也将通过生成的填充程序提供给JS. 如果我们看一下生成的JS代码,我们会看到: 

```js
import * as wasm from './js_hello_world_bg';

export class Foo {
    static __construct(ptr) {
        return new Foo(ptr);
    }

    constructor(ptr) {
        this.ptr = ptr;
    }

    free() {
        const ptr = this.ptr;
        this.ptr = 0;
        wasm.__wbg_foo_free(ptr);
    }

    static new(arg0) {
        const ret = wasm.foo_new(arg0);
        return Foo.__construct(ret)
    }

    get() {
        const ret = wasm.foo_get(this.ptr);
        return ret;
    }

    set(arg0) {
        const ret = wasm.foo_set(this.ptr, arg0);
        return ret;
    }
}
```

那实际上并不多!我们可以在这里看到我们如何从Rust转换为JS: 

-   Rust中的相关函数 (没有`self`)  变成`static`JS中的函数. 
-   Rust中的方法变成了wasm中的方法. 
-   手动内存管理也在JS中公开. 该`free`需要调用函数来释放Rust方面的资源. 

能够使用`new Foo()`,你需要注释`new`如`#[wasm_bindgen(constructor)]`. 

但是,这里需要注意的一个重要方面是`free`被称为JS对象的是"neutered",因为它的内部指针被清空了. 这意味着将来使用此对象应该会在Rust中引发恐慌. 

这些绑定的真正诡计最终会发生在Rust中,所以让我们来看看. 

```rust
// original input to `#[wasm_bindgen]` omitted ...

#[export_name = "foo_new"]
pub extern fn __wasm_bindgen_generated_Foo_new(arg0: i32) -> u32
    let ret = Foo::new(arg0);
    Box::into_raw(Box::new(WasmRefCell::new(ret))) as u32
}

#[export_name = "foo_get"]
pub extern fn __wasm_bindgen_generated_Foo_get(me: u32) -> i32 {
    let me = me as *mut WasmRefCell<Foo>;
    wasm_bindgen::__rt::assert_not_null(me);
    let me = unsafe { &*me };
    return me.borrow().get();
}

#[export_name = "foo_set"]
pub extern fn __wasm_bindgen_generated_Foo_set(me: u32, arg1: i32) {
    let me = me as *mut WasmRefCell<Foo>;
    ::wasm_bindgen::__rt::assert_not_null(me);
    let me = unsafe { &*me };
    me.borrow_mut().set(arg1);
}

#[no_mangle]
pub unsafe extern fn __wbindgen_foo_free(me: u32) {
    let me = me as *mut WasmRefCell<Foo>;
    wasm_bindgen::__rt::assert_not_null(me);
    (*me).borrow_mut(); // ensure no active borrows
    drop(Box::from_raw(me));
}
```

和以前一样,这是从实际输出中清除的,但它与发生的事情是一样的想法!在这里,我们可以看到每个函数的垫片以及用于解除分配实例的垫片`Foo`. 回想一下,今天唯一有效的主义类型是数字,所以我们需要对所有人进行调整`Foo`变成一个`u32`,目前通过`Box` (喜欢`std::unique_ptr`在C ++) . 但请注意,这里有一个额外的图层,`WasmRefCell`. 这种类型与[`RefCell`]并且可以大部分被掩盖. 

如果你感兴趣的话,这种类型的目的是在一个混乱猖獗的世界 (JS) 中维护Rust对别名的保证. 具体是`&Foo`类型意味着可以有你想要的尽可能多的alaising,但至关重要`&mut Foo`意味着它是唯一指向数据的指针 (没有其他指针) `&Foo`到同一个实例存在) . 该[`RefCell`]libstd中的type是一种在运行时动态强制执行此操作的方法 (与通常发生的编译时相反) . 烘烤`WasmRefCell`在这里是相同的想法,添加通常在编译时发生的别名的运行时检查. 这是目前特定于Rust的功能,实际上并不属于`wasm-bindgen`工具本身,它只是在Rust生成的代码 (又名`#[wasm_bindgen]`属性) . 

[`refcell`]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
