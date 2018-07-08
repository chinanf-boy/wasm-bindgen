
# 从JS导入函数

现在我们已经向JS导出了一些丰富的功能,现在也是时候导入一些了!这里的目标是基本实现JS`import`Rust中的语句,具有花哨的类型和所有. 

首先,假设我们反转上面的函数,而不是想在JS中生成问候语,但是从Rust调用它. 例如,我们可能会: 

```rust
#[wasm_bindgen(module = "./greet")]
extern {
    fn greet(a: &str) -> String;
}

fn other_code() {
    let greeting = greet("foo");
    // ...
}
```

导入的基本思想与导出相同,因为我们在JS和Rust中都有垫片进行必要的翻译. 让我们首先看看JS Shim的实际应用: 

```js
import * as wasm from './foo_bg';

import { greet } from './greet';

// ...

export function __wbg_f_greet(ptr0, len0, wasmretptr) {
  const [retptr, retlen] = passStringToWasm(greet(getStringFromWasm(ptr0, len0)));
  (new Uint32Array(wasm.memory.buffer))[wasmretptr / 4] = retlen;
  return retptr;
}
```

该`getStringFromWasm`和`passStringToWasm`和我们之前看到的一样,和我一样`__wbindgen_object_drop_ref`远远高于我们现在从我们的模块出来的这个奇怪的出口!该`__wbg_f_greet`功能是由...产生的`wasm-bindgen`实际上是进口的`foo.wasm`模块. 

生成的`foo.js`我们看到从中进口`./greet`模块与`greet`name (是Rust中的函数导入说) 然后是`__wbg_f_greet`功能是填充导入. 

这里有一些棘手的ABI业务,所以让我们来看看生成的Rust. 像之前一样简化了实际生成的内容. 

```rust
extern fn greet(a: &str) -> String {
    extern {
        fn __wbg_f_greet(a_ptr: *const u8, a_len: usize, ret_len: *mut usize) -> *mut u8;
    }
    unsafe {
        let a_ptr = a.as_ptr();
        let a_len = a.len();
        let mut __ret_strlen = 0;
        let mut __ret_strlen_ptr = &mut __ret_strlen as *mut usize;
        let _ret = __wbg_f_greet(a_ptr, a_len, __ret_strlen_ptr);
        String::from_utf8_unchecked(
            Vec::from_raw_parts(_ret, __ret_strlen, __ret_strlen)
        )
    }
}
```

在这里我们可以看到`greet`函数是生成的,但它主要只是一个垫片`__wbg_f_greet`我们正在打电话的功能. 参数的ptr / len对作为两个参数传递,对于返回值,我们在直接接收返回的指针时间接接收一个值 (长度) . 
