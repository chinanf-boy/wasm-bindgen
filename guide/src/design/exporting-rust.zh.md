
# 将函数导出到JS

好了,现在我们已经很好地掌握了JS对象以及它们是如何工作的,让我们来看看另一个特性. `wasm-bindgen`: 使用比数字更丰富的类型导出功能. 

导出具有更多美味类型的功能的基本思想是,实际上不会直接调用wasm导出. 而是生成的`foo.js`模块将为wasm模块中的所有导出函数提供填充程序. 

这里最有趣的转换发生在字符串上,让我们来看看. 

```rust
#[wasm_bindgen]
pub fn greet(a: &str) -> String {
    format!("Hello, {}!", a)
}
```

在这里,我们要定义一个看起来像的ES模块

```ts
// foo.d.ts
export function greet(a: string): string;
```

要了解发生了什么,让我们看一下生成的垫片

```js
import * as wasm from './foo_bg';

function passStringToWasm(arg) {
  const buf = new TextEncoder('utf-8').encode(arg);
  const len = buf.length;
  const ptr = wasm.__wbindgen_malloc(len);
  let array = new Uint8Array(wasm.memory.buffer);
  array.set(buf, ptr);
  return [ptr, len];
}

function getStringFromWasm(ptr, len) {
  const mem = new Uint8Array(wasm.memory.buffer);
  const slice = mem.slice(ptr, ptr + len);
  const ret = new TextDecoder('utf-8').decode(slice);
  return ret;
}

export function greet(arg0) {
  const [ptr0, len0] = passStringToWasm(arg0);
  try {
    const ret = wasm.greet(ptr0, len0);
    const ptr = wasm.__wbindgen_boxed_str_ptr(ret);
    const len = wasm.__wbindgen_boxed_str_len(ret);
    const realRet = getStringFromWasm(ptr, len);
    wasm.__wbindgen_boxed_str_free(ret);
    return realRet;
  } finally {
    wasm.__wbindgen_free(ptr0, len0);
  }
}
```

哎呀,那真是太多了!如果仔细观察发生了什么,我们可以看一看: 

-   字符串通过两个参数 (指针和长度) 传递给wasm. 现在我们必须将字符串复制到wasm堆上,这意味着我们将使用它`TextEncoder`实际上做编码. 完成后,我们使用内部函数`wasm-bindgen`为字符串分配空间,然后我们将稍后将该ptr / length传递给wasm. 

-   从wasm返回字符串有点棘手,因为我们需要返回一个ptr / len对,但ism目前只支持一个返回值 (多个返回值) [正在标准化](https://github.com/WebAssembly/design/issues/1146)) . 为了解决这个问题,我们实际上返回一个指向ptr / len对的指针,然后使用函数来访问各个字段. 

-   一些清理工作最终会在wasm中发生. 该`__wbindgen_boxed_str_free`function用于释放返回值`greet`在它被解码到JS堆上之后 (使用`TextDecoder`) . 该`__wbindgen_free`然后,在函数调用完成后,用于释放我们分配的空间以传递字符串参数. 

接下来让我们来看看Rust的一面. 在这里,我们将看到一个大部分缩写和/或"简化"的意义,它是由它编写的: 

```rust
pub extern fn greet(a: &str) -> String {
    format!("Hello, {}!", a)
}

#[export_name = "greet"]
pub extern fn __wasm_bindgen_generated_greet(
    arg0_ptr: *const u8,
    arg0_len: usize,
) -> *mut String {
    let arg0 = unsafe {
        let slice = ::std::slice::from_raw_parts(arg0_ptr, arg0_len);
        ::std::str::from_utf8_unchecked(slice)
    };
    let _ret = greet(arg0);
    Box::into_raw(Box::new(_ret))
}
```

在这里,我们可以再次看到我们的`greet`函数是未修改的,并有一个包装器来调用它. 这个包装器将获取ptr / len参数并将其转换为字符串切片,而返回值仅被装入一个指针,然后返回到通过`__wbindgen_boxed_str_*`功能. 

因此,一般情况下,导出函数涉及JS和Rust中的垫片,每一侧都转换为每种语言的本机类型的ism参数. 该`wasm-bindgen`工具管理连接所有这些垫片`#[wasm_bindgen]`宏也照顾Rust垫片. 

大多数论点都有一个相对清晰的方法来转换它们,如果你有任何问题请告诉我!
