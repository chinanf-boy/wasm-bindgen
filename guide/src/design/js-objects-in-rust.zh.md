
# Polyfill for"JS objects in wasm"

其中一个主要目标是`wasm-bindgen`是允许在wasm中使用和传递JS对象,但今天不允许这样做!虽然确实如此,但这就是polyfill的用武之地. 

这里的问题是我们如何将JS对象鞋拔成一个`u32`对于使用ism. 此方法的当前策略是在生成的中维护两个模块局部变量`foo.js`file: 堆栈和堆. 

### 堆栈上的临时JS对象

堆栈中`foo.js`是,堆栈. JS对象被推送到堆栈的顶部,它们在堆栈中的索引是传递给wasm的标识符. 然后,JS对象也只从堆栈顶部移除. 这种数据结构主要用于有效地将JS对象传递给wasm而不需要"堆分配". 然而,它的缺点是它只适用于当wasm不能保留JS对象时 (也就是说它只能用Rust的说法获得"引用") . 

我们来看一个例子. 

```rust
// foo.rs
#[wasm_bindgen]
pub fn foo(a: &JsValue) {
    // ...
}
```

我们在这里使用特殊的`JsValue`从中输入`wasm-bindgen`图书馆本身. 我们的出口功能,`foo`拿一个*参考*到一个对象. 这显然意味着它不能将对象持久化超过此函数调用的生命周期. 

现在我们实际想要生成的是一个看起来像的JS模块 (在Typescript用语中) 

```ts
// foo.d.ts
export function foo(a: any);
```

我们实际生成的东西看起来像: 

```js
// foo.js
import * as wasm from './foo_bg';

let stack = [];

function addBorrowedObject(obj) {
  stack.push(obj);
  return stack.length - 1;
}

export function foo(arg0) {
  const idx0 = addBorrowedObject(arg0);
  try {
    wasm.foo(idx0);
  } finally {
    stack.pop();
  }
}
```

在这里,我们可以看到一些值得注意的行动点: 

-   wasm文件已重命名为`foo_bg.wasm`,我们可以看到这里生成的JS模块是如何从wasm文件导入的. 
-   接下来我们可以看到我们`stack`模块变量,用于从堆栈中推送/弹出项目. 
-   我们的出口功能`foo`,任意争论,`arg0`,转换为索引`addBorrowedObject`对象功能. 然后将索引传递给wasm,因此ism可以使用它. 
-   最后,我们有一个`finally`它释放堆栈槽,因为它不再使用,发出一个`pop`对于在函数开始时推送的内容. 

挖掘Rust的一面也很有帮助,看看那里发生了什么!我们来看看那些代码`#[wasm_bindgen]`在Rust生成: 

```rust
// what the user wrote
pub fn foo(a: &JsValue) {
    // ...
}

#[export_name = "foo"]
pub extern fn __wasm_bindgen_generated_foo(arg0: u32) {
    let arg0 = unsafe {
        ManuallyDrop::new(JsValue::__from_idx(arg0))
    };
    let arg0 = &*arg0;
    foo(arg0);
}
```

和JS一样,这里值得注意的要点是: 

-   原来的功能,`foo`,在输出中未经修改
-   这里生成的函数 (具有唯一名称) 是实际从wasm模块导出的函数
-   我们生成的函数接受一个整数参数 (我们的索引) ,然后将其包装在一个`JsValue`. 这里有一些不值得进入的诡计,但我们会稍微看一下发生在幕后的事情. 

### 板中长寿的JS对象

当JS对象仅在Rust中临时使用时,上述策略很有用,例如仅在一次函数调用期间. 但有时,对象可能具有动态生命周期,或者需要存储在Rust的堆上. 为了解决这个问题,JS对象的后半部分是一个平板. 

传递给wasm的JS对象不是引用,假定在wasm模块内部具有动态生命周期. 因此,堆栈的严格推送/弹出将不起作用,我们需要更多的JS对象永久存储. 为了应对这种情况,我们建立了自己的"板块分配器". 

一张图片 (或代码) 值得一千个字,所以让我们展示一个例子会发生什么. 

```rust
// foo.rs
#[wasm_bindgen]
pub fn foo(a: JsValue) {
    // ...
}
```

请注意`&`在前面缺少`JsValue`我们之前有过,而在Rust的说法中,这意味着它取得了JS值的所有权. 导出的ES模块接口与以前相同,但所有权机制略有不同. 让我们看一下生成的JS板块: 

```js
import * as wasm from './foo_bg'; // imports from wasm file

let slab = [];
let slab_next = 0;

function addHeapObject(obj) {
  if (slab_next === slab.length)
    slab.push(slab.length + 1);
  const idx = slab_next;
  const next = slab[idx];
  slab_next = next;
  slab[idx] = { obj, cnt: 1 };
  return idx;
}

export function foo(arg0) {
  const idx0 = addHeapObject(arg0);
  wasm.foo(idx0);
}

export function __wbindgen_object_drop_ref(idx) {
  let obj = slab[idx];
  obj.cnt -= 1;
  if (obj.cnt > 0)
    return;
  // If we hit 0 then free up our space in the slab
  slab[idx] = slab_next;
  slab_next = idx;
}
```

不像以前我们现在打电话`addHeapObject`关于这个论点`foo`而不是`addBorrowedObject`. 这个功能会用到`slab`和`slab_next`作为slab分配器获取存储对象的槽,一旦找到它就放置一个结构. 

注意,除了存储对象之外,还使用引用计数. 这样我们就可以在不使用的情况下在Rust中创建对JS对象的多个引用`Rc`,但总的来说,担心这一点并不重要. 

这个生成的模块的另一个奇怪的方面是`__wbindgen_object_drop_ref`功能. 这是一个实际上是从wasm导入而不是在这个模块中使用的!此函数用于表示a的生命周期结束`JsValue`在Rust中,或者换句话说,当它超出范围时. 否则,虽然这个功能很大程度上只是一个普通的"无板"实现. 

最后,让我们再看一下Rust生成的内容: 

```rust
// what the user wrote
pub fn foo(a: JsValue) {
    // ...
}

#[export_name = "foo"]
pub extern fn __wasm_bindgen_generated_foo(arg0: u32) {
    let arg0 = unsafe {
        JsValue::__from_idx(arg0)
    };
    foo(arg0);
}
```

啊,看起来更熟悉!这里没有太多有趣的事情发生,所以让我们继续......

### 解剖学`JsValue`

目前`JsValue`在Rust中,struct实际上非常简单,它是: 

```rust
pub struct JsValue {
    idx: u32,
}

// "private" constructors

impl Drop for JsValue {
    fn drop(&mut self) {
        unsafe {
            __wbindgen_object_drop_ref(self.idx);
        }
    }
}
```

或者换句话说,它是一个新的类型包装器`u32`,我们从ism传递的索引. 这里的析构函数就是这里的`__wbindgen_object_drop_ref`函数被调用以放弃我们对JS对象的引用计数,从而释放我们的插槽`slab`我们在上面看到了. 

如果你还记得,当我们采取`&JsValue`上面我们生成了一个包装器`ManuallyDrop`围绕本地绑定,这是因为我们想避免在对象来自堆栈时调用此析构函数. 

### 索引板和堆栈

您可能在想这个系统可能不起作用!平板和堆栈的索引混合在一起,但我们如何区分?事实证明,上面的例子已经简化了一些,但是最低位当前用作指示你是slab还是堆栈索引. 
