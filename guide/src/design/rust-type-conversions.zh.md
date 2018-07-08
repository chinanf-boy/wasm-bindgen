
# Rust类型转换

以前,当值输入Rust时,我们已经看到大多数类型转换的删节版本. 在这里,我们将深入探讨如何实现这一点. 转换值有两类特征,将值从Rust转换为JS的特性和反过来的特征. 

### 从Rust到JS

首先让我们来看看从Rust到JS: 

```rust
pub trait IntoWasmAbi: WasmDescribe {
    type Abi: WasmAbi;
    fn into_abi(self, extra: &mut Stack) -> Self::Abi;
}
```

就是这样!这实际上是目前将Rust值转换为JS值所需的唯一特征. 这里有几点: 

-   我们会去的`WasmDescribe`在本节后面
-   相关类型`Abi`是实际生成的作为wasm导出的参数. 界限`WasmAbi`仅适用于类似的类型`u32`和`f64`那些可以放在边界上并无损传输的. 
-   最后我们有了`into_abi`功能,返回`Abi`实际传递给JS的关联类型. 还有这个`Stack`然而,参数. 并非所有Rust值都可以32位通信`Stack`参数允许传输更多数据,稍后解释. 

此特征适用于所有可转换为JS的类型,并且在codegen期间无条件使用. 例如,你经常会看到`IntoWasmAbi
for Foo`但也`IntoWasmAbi for &'a Foo`. 

该`IntoWasmAbi`特征在两个地方使用. 首先,它用于将Rust导出函数的返回值转换为JS. 其次,它用于转换导入到Rust的JS函数的Rust参数. 

### 从JS到Rust

不幸的是,从JS到Rust的上面相反的方向有点复杂. 这里我们有三个特点: 

```rust
pub trait FromWasmAbi: WasmDescribe {
    type Abi: WasmAbi;
    unsafe fn from_abi(js: Self::Abi, extra: &mut Stack) -> Self;
}

pub trait RefFromWasmAbi: WasmDescribe {
    type Abi: WasmAbi;
    type Anchor: Deref<Target=Self>;
    unsafe fn ref_from_abi(js: Self::Abi, extra: &mut Stack) -> Self::Anchor;
}

pub trait RefMutFromWasmAbi: WasmDescribe {
    type Abi: WasmAbi;
    type Anchor: DerefMut<Target=Self>;
    unsafe fn ref_mut_from_abi(js: Self::Abi, extra: &mut Stack) -> Self::Anchor;
}
```

该`FromWasmAbi`相对简单,基本上是相反的`IntoWasmAbi`. 它采用ABI参数 (通常与...相同) `IntoWasmAbi::Abi`) 然后辅助堆栈生成一个实例`Self`. 该特征主要用于那些类型*别*有内部生命或参考. 

这里的后两个特征大部分是相同的,用于生成引用 (共享和可变引用) . 他们看起来几乎一样`FromWasmAbi`除了他们回来了`Anchor`实现的类型`Deref`特质而不是`Self`. 

该`Ref*`例如,traits允许在引用而不是裸类型的函数中使用参数`&str`,`&JsValue`, 要么`&[u8]`. 该`Anchor`这里需要确保生命周期不会超出一个函数调用并保持匿名. 

该`From*`traistic系列用于将Rust导出函数中的Rust参数转换为JS. 它们还用于导入Rust的JS函数的返回值. 

### 全球堆栈

上面提到的并非所有Rust类型都适合32位. 虽然我们可以沟通`f64`我们不一定能够使用所有的比特. 类似的`&str`需要传达两个项目,一个指针和一个长度 (64位) . 其他类型如`&Closure<Fn()>`有更多的信息要传输. 

因此,我们需要一种通过函数签名传递更多数据的方法. 虽然我们可以添加更多的参数,但在闭包世界中这有点难以做到,因为代码生成并不像程序宏那样动态. 因此,"全局堆栈"用于传输函数调用的额外数据. 

全局堆栈是wasm模块中固定大小的静态分配. 这个堆栈是从JS到Rust或Rust ot JS的任何一个函数调用的临时临时空间. Rust和生成的JS填充程序都有指向此全局堆栈的指针,并将从中读取/写入信息. 

每当我们想通过时使用这个方案`&str`从JS到Rust,我们可以将指针作为实际的ABI参数传递,然后将长度放在全局堆栈的下一个位置. 

该`Stack`上面转换特征的参数如下所示: 

```rust
pub trait Stack {
    fn push(&mut self, bits: u32);
    fn pop(&mut self) -> u32;
}
```

这里使用特征来促进测试,但通常调用最终不会在运行时被虚拟调度. 
