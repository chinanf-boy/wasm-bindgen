
# 我们还能做什么?

多得多! 这里介绍了您可以在此项目中使用的各种功能. 你也可以[在线探索此代码](https://webassembly.studio/?f=t61j18noqz): 

``` rust
// src/lib.rs
#![feature(proc_macro, wasm_custom_section, wasm_import_module)]

extern crate wasm_bindgen;

use wasm_bindgen::prelude::*;

// Strings can both be passed in and received
#[wasm_bindgen]
pub fn concat(a: &str, b: &str) -> String {
    let mut a = a.to_string();
    a.push_str(b);
    return a
}

// A struct will show up as a class on the JS side of things
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

    // Methods can be defined with `&mut self` or `&self`, and arguments you
    // can pass to a normal free function also all work in methods.
    pub fn add(&mut self, amt: u32) -> u32 {
        self.contents += amt;
        return self.contents
    }

    // You can also take a limited set of references to other types as well.
    pub fn add_other(&mut self, bar: &Bar) {
        self.contents += bar.contents;
    }

    // Ownership can work too!
    pub fn consume_other(&mut self, bar: Bar) {
        self.contents += bar.contents;
    }
}

#[wasm_bindgen]
pub struct Bar {
    contents: u32,
    opaque: JsValue, // defined in `wasm_bindgen`, imported via prelude
}

#[wasm_bindgen(module = "./index")] // what ES6 module to import from
extern {
    fn bar_on_reset(to: &str, opaque: &JsValue);

    // We can import classes and annotate functionality on those classes as well
    type Awesome;
    #[wasm_bindgen(constructor)]
    fn new() -> Awesome;
    #[wasm_bindgen(method)]
    fn get_internal(this: &Awesome) -> u32;
}

#[wasm_bindgen]
impl Bar {
    pub fn from_str(s: &str, opaque: JsValue) -> Bar {
        let contents = s.parse().unwrap_or_else(|_| {
            Awesome::new().get_internal()
        });
        Bar { contents, opaque }
    }

    pub fn reset(&mut self, s: &str) {
        if let Ok(n) = s.parse() {
            bar_on_reset(s, &self.opaque);
            self.contents = n;
        }
    }
}
```

此宏调用,生成的JS绑定[看起来像这样 `翻墙`][bindings]. 你可以像这样查看它们: 

[bindings]: https://gist.github.com/alexcrichton/3d85c505e785fb8ff32e2c1cf9618367

<details>

<summary> 生成的JS绑定 </summary>

``` js
/* tslint:disable */
import * as wasm from './js_hello_world_wasm'; // imports from wasm file

import {
    bar_on_reset
} from './index';

import {
    Awesome
} from './index';

let cachedEncoder = null;

function textEncoder() {
    if (cachedEncoder)
        return cachedEncoder;
    cachedEncoder = new TextEncoder('utf-8');
    return cachedEncoder;
}

let cachedUint8Memory = null;

function getUint8Memory() {
    if (cachedUint8Memory === null ||
        cachedUint8Memory.buffer !== wasm.memory.buffer)
        cachedUint8Memory = new Uint8Array(wasm.memory.buffer);
    return cachedUint8Memory;
}

function passStringToWasm(arg) {
    if (typeof(arg) !== 'string')
        throw new Error('expected a string argument');
    const buf = textEncoder().encode(arg);
    const len = buf.length;
    const ptr = wasm.__wbindgen_malloc(len);
    getUint8Memory().set(buf, ptr);
    return [ptr, len];
}

let slab = [];
let slab_next = 0;

function addHeapObject(obj) {
    if (slab_next == slab.length)
        slab.push(slab.length + 1);
    const idx = slab_next;
    const next = slab[idx];
    slab_next = next;
    slab[idx] = {
        obj,
        cnt: 1
    };
    return idx << 1;
}

let cachedDecoder = null;

function textDecoder() {
    if (cachedDecoder)
        return cachedDecoder;
    cachedDecoder = new TextDecoder('utf-8');
    return cachedDecoder;
}

function getStringFromWasm(ptr, len) {
    const mem = getUint8Memory();
    const slice = mem.slice(ptr, ptr + len);
    const ret = textDecoder().decode(slice);
    return ret;
}

let stack = [];

function getObject(idx) {
    if ((idx & 1) === 1) {
        return stack[idx >> 1];
    } else {
        const val = slab[idx >> 1];

        return val.obj;

    }
}

export function __wbg_f_bar_on_reset(ptr0, len0, arg1) {
    bar_on_reset(getStringFromWasm(ptr0, len0), getObject(arg1))
}

export function __wbg_s_Awesome_new() {
    return addHeapObject(new Awesome());
}

export function __wbg_s_Awesome_get_internal(arg0) {
    return Awesome.prototype.get_internal.call(getObject(arg0));
}

export function concat(arg0, arg1) {
    const [ptr0, len0] = passStringToWasm(arg0);
    const [ptr1, len1] = passStringToWasm(arg1);
    try {
        const ret = wasm.concat(ptr0, len0, ptr1, len1);

        const ptr = wasm.__wbindgen_boxed_str_ptr(ret);
        const len = wasm.__wbindgen_boxed_str_len(ret);
        const realRet = getStringFromWasm(ptr, len);
        wasm.__wbindgen_boxed_str_free(ret);
        return realRet;

    } finally {

        wasm.__wbindgen_free(ptr0, len0);

        wasm.__wbindgen_free(ptr1, len1);

    }
}

export class Foo {
    constructor(ptr) {
        this.ptr = ptr;
    }

    free() {
        const ptr = this.ptr;
        this.ptr = 0;
        wasm.__wbg_foo_free(ptr);
    }
    static new() {
        const ret = wasm.foo_new();
        return new Foo(ret);

    }
    add(arg0) {
        const ret = wasm.foo_add(this.ptr, arg0);
        return ret;
    }
    add_other(arg0) {
        const ret = wasm.foo_add_other(this.ptr, arg0.ptr);
        return ret;
    }
    consume_other(arg0) {
        const ptr0 = arg0.ptr;
        arg0.ptr = 0;
        const ret = wasm.foo_consume_other(this.ptr, ptr0);
        return ret;
    }
}

export class Bar {
    constructor(ptr) {
        this.ptr = ptr;
    }

    free() {
        const ptr = this.ptr;
        this.ptr = 0;
        wasm.__wbg_bar_free(ptr);
    }
    static from_str(arg0, arg1) {
        const [ptr0, len0] = passStringToWasm(arg0);
        const idx1 = addHeapObject(arg1);
        try {
            const ret = wasm.bar_from_str(ptr0, len0, idx1);
            return new Bar(ret);

        } finally {

            wasm.__wbindgen_free(ptr0, len0);

        }
    }
    reset(arg0) {
        const [ptr0, len0] = passStringToWasm(arg0);
        try {
            const ret = wasm.bar_reset(this.ptr, ptr0, len0);
            return ret;
        } finally {

            wasm.__wbindgen_free(ptr0, len0);

        }
    }
}

function dropRef(idx) {
    let obj = slab[idx >> 1];
    obj.cnt -= 1;
    if (obj.cnt > 0)
        return;
    // If we hit 0 then free up our space in the slab
    slab[idx >> 1] = slab_next;
    slab_next = idx >> 1;
}

export const __wbindgen_object_drop_ref = dropRef;

export const __wbindgen_throw =
    function(ptr, len) {
        throw new Error(getStringFromWasm(ptr, len));
    };
```

</details>

- 

和我们使用它,在`index.js`: 

```js
import { Foo, Bar, concat } from "./js_hello_world";
import { booted } from "./js_hello_world_wasm";

export function bar_on_reset(s, token) {
  console.log(token);
  console.log(`this instance of bar was reset to ${s}`);
}

function assertEq(a, b) {
  if (a !== b)
    throw new Error(`${a} != ${b}`);
  console.log(`found ${a} === ${b}`);
}

function main() {
  assertEq(concat('a', 'b'), 'ab');

  // Note that to use `new Foo()` the constructor function must be annotated
  // with `#[wasm_bindgen(constructor)]`, otherwise only `Foo.new()` can be used.
  // Additionally objects allocated corresponding to Rust structs will need to
  // be deallocated on the Rust side of things with an explicit call to `free`.
  let foo = new Foo();
  assertEq(foo.add(10), 10);
  foo.free();

  // Pass objects to one another
  let foo1 = new Foo();
  let bar = Bar.from_str("22", { opaque: 'object' });
  foo1.add_other(bar);

  // We also don't have to `free` the `bar` variable as this function is
  // transferring ownership to `foo1`
  bar.reset('34');
  foo1.consume_other(bar);

  assertEq(foo1.add(2), 22 + 34 + 2);
  foo1.free();

  alert('all passed!')
}

export class Awesome {
  constructor() {
    this.internal = 32;
  }

  get_internal() {
    return this.internal;
  }
}

booted.then(main);
```
