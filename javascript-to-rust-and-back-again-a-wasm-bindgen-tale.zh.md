## 从`js`到`rust`再回来: `wasm-bindgen`的故事

最近我们已经看到了`WebAssembly`是怎样的[编译速度极快](https://hacks.mozilla.org/2018/01/making-webassembly-even-faster-firefoxs-new-streaming-and-tiering-compiler/),[加快 JS 库的速度](https://hacks.mozilla.org/2018/01/oxidizing-source-maps-with-rust-and-webassembly/),和[生成更小的二进制文件](https://hacks.mozilla.org/2018/01/shrinking-webassembly-and-javascript-code-sizes-in-emscripten/). 我们甚至[得到了一个高层次的计划](https://hacks.mozilla.org/2018/03/making-webassembly-better-for-rust-for-all-languages/)为了更好地实现 Rust 和 JavaScript 社区 之间的互操作性,以及其他 Web 编程语言. 正如[上一篇文章](https://hacks.mozilla.org/2018/03/making-webassembly-better-for-rust-for-all-languages/)提到的那样, 我想深入了解一个特定组件的更多细节,那就是[`wasm-bindgen`](https://github.com/alexcrichton/wasm-bindgen).

今天[WebAssembly 规范](https://webassembly.github.io/spec/)仅定义了四种类型: 两种整数类型和两种浮点类型. 但是,大多数情况下,JS 和 Rust 开发人员正在使用更丰富的类型! 例如,JS 开发人员经常与[`document`](https://developer.mozilla.org/en-US/docs/Web/API/Document)交互, 添加或修改 HTML 节点,而 Rust 开发人员使用类似的类型[`Result`](https://doc.rust-lang.org/std/result/enum.Result.html)用于错误处理,几乎所有程序员都使用字符串.

限于今天 WebAssembly 所提供的,类型将限制太多,这是缺点之一

`wasm-bindgen`进入了画面. `wasm-bindgen`的目标是提供 JS 和 Rust 类型之间的桥梁. 它允许 JS 使用字符串调用 Rust API,或者使用 Rust 函数来捕获 JS 异常. `wasm-bindgen`擦除 WebAssembly 与 JavaScript 之间不匹配的阻抗,确保 JavaScript 可以有效地调用 WebAssembly 函数 而无需样板,并且 WebAssembly 可以对 JavaScript 函数 执行相同的操作.

该`wasm-bindgen`项目中有更多的描述[自述](./README.md). 要开始使用,我们来看一个使用示例`wasm-bindgen`, 然后探索它还能提供什么.

## Hello, World!

始终是经典,学习新工具的最佳方法之一, 是探索其相当于打印"你好,世界!" 在这种情况下,我们将探索一个[例子](https://github.com/alexcrichton/wasm-bindgen/tree/master/examples/hello_world) - alert "你好,世界!" 到页面.

这里的目标很简单,我们想定义 一个 Rust 函数,给定一个名称,将在页面上创建一个对话框`Hello, ${name}!`. 在 JavaScript 中,我们可以将此函数定义为:

    export function greet(name) {
        alert(`Hello, ${name}!`);
    }

不过,这个例子的 alert , 我们想把它写在 Rust 中. 在这里已经发生了许多, 我们必须要处理的事情:

- JavaScript 将调用 WebAssembly 模块,即`greet` export.
- Rust 函数将 字符串 作为输入,`name`被`greet`使用
- 内部 Rust 将创建 一个插入内部名称的新字符串.
- 最后 Rust 会调用 JavaScript[`alert`](https://developer.mozilla.org/en-US/docs/Web/API/Window/alert)函数, 还有创建的字符串.

首先,我们创建一个全新的 Rust 项目:

    $ cargo new wasm-greet --lib

这在我们工作的文件夹里面, 初始化了一个新的`wasm-greet`. 接下来我们修改我们的`Cargo.toml` (相当于`package.json`对于 Rust) ,包含以下信息:

    [lib]
    crate-type = ["cdylib"]

    [dependencies]
    wasm-bindgen = "0.2"

我们忽略了`[lib]`现在的业务,但下一部分声明依赖于[`wasm-bindgen`箱子](https://crates.io/crates/wasm-bindgen). 箱子包含我们需要在 Rust 使用的`wasm-bindgen`所有支持.

接下来,是时候写一些代码了!我们替换自动创建的`src/lib.rs`这些内容:

    #![feature(proc_macro, wasm_custom_section, wasm_import_module)]

    extern crate wasm_bindgen;

    use wasm_bindgen::prelude::*;

    #[wasm_bindgen]
    extern {
        fn alert(s: &str);
    }

    #[wasm_bindgen]
    pub fn greet(name: &str) {
        alert(&format!("Hello, {}!", name));
    }

如果你对 Rust 不熟悉,这可能看起来有点罗嗦,但不要害怕! 该`wasm-bindgen`项目随着时间的推移不断改进,而且可以肯定的是,所有这些并不总是必要的. 最值得注意的是`#[wasm_bindgen]` _属性_, Rust 代码中的注释,这里的意思是 "_请根据需要的使用包装进行处理_" . 我们都进口了`alert`功能和出口`greet`函数使用此属性进行注释. 片刻之后,我们将看到幕后发生的事情.

但首先,我们切入, 并在浏览器中打开它! 让我们编译我们的`wasm`代码:

    $ rustup target add wasm32-unknown-unknown --toolchain nightly # only needed once
    $ cargo +nightly build --target wasm32-unknown-unknown

这给了我们一个`wasm`文件`target/wasm32-unknown-unknown/debug/wasm_greet.wasm`. 如果我们使用像[`wasm2wat`](https://github.com/WebAssembly/wabt)这样的工具查看这个`wasm`文件, 它可能有点可怕. 事实证明这个`wasm`文件实际上还没准备好被 JS 消费了! 相反,我们还需要一个步骤来使其可用:

    $ cargo install wasm-bindgen-cli # only needed once
    $ wasm-bindgen target/wasm32-unknown-unknown/debug/wasm_greet.wasm --out-dir .

这一步是很多魔术发生的地方: `wasm-bindgen`CLI 工具后处理输入`wasm`文件,使其被合适使用. 稍后我们会看到"合适"意味着什么,现在只要我们导入新创建的就足够了`wasm_greet.js`文件 (由`wasm-bindgen`工具创建) 我们得到了`greet` - 我们在 Rust 中定义的函数.

最后,我们要做的就是用 捆绑器 打包它,并创建一个 HTML 页面来运行我们的代码. 在撰写本文时,仅限于[Webpack 的 4.0 版本](https://medium.com/webpack/webpack-4-released-today-6cdb994702d4)足够支持 WebAssembly 开箱即用 (虽然它也有一个[Chrome alert ](https://github.com/alexcrichton/wasm-bindgen/blob/master/examples/hello_world/README.md#caveat-for-chrome-users)暂且) . 随着时间的推移,更多的捆绑商肯定会跟进. 我会在这里略过细节,但你可以关注[例子-hello world](https://github.com/alexcrichton/wasm-bindgen/tree/master/examples/hello_world)Github repo 中的配置. 如果我们查看内部,我们的页面 JS 看起来像:

    const rust = import("./wasm_greet");
    rust.then(m => m.greet("World!"));

...就是这样! 打开我们的网页现在应该显示一个很好的"Hello,World!"对话框,由 Rust 驱动.

## `wasm-bindgen` 如何工作

[Phew](#phew),这是一个有个大点的"Hello,World!", 让我们深入了解一下底下发生的事情以及工具的工作原理.

最重要的一个方面,它的集成基本上建立在一个概念上,即`wasm`模块只是另一种 ES 模块. 例如,我们*想*带有签名的 (在 Typescript 中) 的 ES 模块,如下所示:

    export function greet(s: string);

原生 WebAssembly 没有办法这样做 (请记住它今天只支持数字) ,所以我们依赖`wasm-bindgen`填补空白. 在我们上面的最后一步,当我们跑了`wasm-bindgen`工具你会注意到一个`wasm_greet.js`文件和一起发出的`wasm_greet_bg.wasm`文件. 前者是我们想要的实际 JS 接口,执行任何必要的粘合来调用 Rust. 该`*_bg.wasm`file 包含实际的实现 和 我们所有编译的代码.

当我们导入`./wasm_greet`时, 我们得到 Rust 代码*可以*暴露的模块, 但今天本身不能这样做. 现在我们已经看到了集成的工作原理,让我们按照脚本的执行情况来看看会发生什么. 首先,我们的示例运行:

    const rust = import("./wasm_greet");
    rust.then(m => m.greet("World!"));

在这里,我们异步导入我们想要的接口,等待它解决 (通过下载和编译`wasm`) . 然后我们运行模块上`greet`的函数.

> **注意**: 这里的异步加载是[需要新的 Webpack](https://github.com/webpack/webpack/issues/6615),但这可能并非总是如此,可能不是其他捆绑商的要求.

如果我们看看里面的`wasm_greet.js`文件. `wasm-bindgen`工具生成就像:

    import * as wasm from './wasm_greet_bg';

    // ...

    export function greet(arg0) {
        const [ptr0, len0] = passStringToWasm(arg0);
        try {
            const ret = wasm.greet(ptr0, len0);
            return ret;
        } finally {
            wasm.__wbindgen_free(ptr0, len0);
        }
    }

    export function __wbg_f_alert_alert_n(ptr0, len0) {
        // ...
    }

> **注意**: 请记住,这是未经优化和生成的代码,它可能不是很好或很小! 当使用 LTO (链接时间优化) 和 Rust 中的发布版本编译时,以及在通过 JS bundler 管道 (缩小) 之后,这应该小得多.

在这里我们可以看到`wasm-bindgen`正在生成我们的`greet`函数. 在引擎盖下,它仍然在呼唤着`wasm`的`greet`函数,但它用指针`ptr0`和长度`len0`而不是字符串调用. 有关的更多细节`passStringToWasm`你可以看到[林克拉克之前的帖子](https://hacks.mozilla.org/2018/03/making-webassembly-better-for-rust-for-all-languages/). 这都是样板代码`wasm-bindgen`工具会为我们照顾它! 我们得到的`__wbg_f_alert_alert_n`马上起作用.

更深层次的移动,下一个感兴趣的项目是 WebAssembly 中的`greet`函数. 为了查看它,让我们看看 Rust 编译器看到的代码. 请注意,就像上面生成的 JS 包装器 一样,你不是在这里编写`greet`输出符号,而是`#[wasm_bindgen]`属性正在生成一个垫片,它为您进行翻译,即:

    pub fn greet(name: &str) {
        alert(&format!("Hello, {}!", name));
    }

    #[export_name = "greet"]
    pub extern fn __wasm_bindgen_generated_greet(arg0_ptr: *mut u8, arg0_len: usize) {
        let arg0 = unsafe { ::std::slice::from_raw_parts(arg0_ptr as *const u8, arg0_len) }
        let arg0 = unsafe { ::std::str::from_utf8_unchecked(arg0) };
        greet(arg0);
    }

在这里,我们可以看到原始代码,`greet`,但是`#[wasm_bingen]`属性插入这个有趣的`__wasm_bindgen_generated_greet`. 这是一个导出的函数 (用`#[export_name]`和`extern`获取 JS 传入的指针/长度对. 然后在内部将指针/长度转换为一个 [`&str`](https://doc.rust-lang.org/std/primitive.str.html)(Rust 中的一个 String) 并转成我们定义的`greet`.

换一种方式, `#[wasm_bindgen]`属性生成两个包装器: 一个在 JavaScript 中,它将 JS 类型转换为 wasm,另一个在 Rust 中,它接收 wasm 类型 并转换为 Rust 类型.

好吧,让我们看看最后一套包装纸, `alert`函数. 该`greet`函数,在 Rust 中的使用标准 - 用于创建新字符串然后将其传递给`alert`的宏[`format!`](https://doc.rust-lang.org/std/macro.format.html). 回想一下,当我们声明`alert`时带有`#[wasm_bindgen]`, 让我们看看 rustc 对此函数的样子:

    fn alert(s: &str) {
        #[wasm_import_module = "__wbindgen_placeholder__"]
        extern {
            fn __wbg_f_alert_alert_n(s_ptr: *const u8, s_len: usize);
        }
        unsafe {
            let s_ptr = s.as_ptr();
            let s_len = s.len();
            __wbg_f_alert_alert_n(s_ptr, s_len);
        }
    }

现在这些都不是我们写的, 但我们可以看到它是如何结合在一起的.
该函数实际上是一个使用 Rust[`&str`](https://doc.rust-lang.org/std/primitive.str.html)的小包装器, 然后将`alert`转换为`wasm`类型 (数字) . 这叫做我们的`__wbg_f_alert_alert_n`和我们在上面看到的有趣功能,但这有个奇怪的`#[wasm_import_module]`属性.

WebAssembly 中的所有函数导入都有一个它们来源的模块,回头看, `wasm-bindgen`是基于 ES 模块构建的,这也将被解释为 ES 模块导入! 现在`__wbindgen_placeholder__`模块实际上并不存在,但它表示此导入将被`wasm-bindgen`重写, 为了从我们生成的 JS 文件导入.

最后,对于拼图的最后一部分,我们得到了生成的 JS 文件,其中包含:

    export function __wbg_f_alert_alert_n(ptr0, len0) {
        let arg0 = getStringFromWasm(ptr0, len0);
        alert(arg0)
    }

哇!事实证明,相当多的事情发生在这里,在 JS 中,我们有一个相对较长的`greet`链到在浏览器中`alert`. 但不要担心,这是`wasm-bindgen`关键, 并且对我们是隐藏所有这些基础设施! 您只需要编写一些 Rust 代码`#[wasm_bindgen]`这里和那里. 然后你的 JS 可以像使用 另一个 JS 包或模块 一样使用 Rust.

### What else can `wasm-bindgen` do?

该`wasm-bindgen`项目范围很广,我们没有时间详细介绍这里的所有细节. 探索`wasm-bindgen`功能的好方法是探索[示例目录](https://github.com/alexcrichton/wasm-bindgen/tree/master/examples), 范围从[你好,世界!](https://github.com/alexcrichton/wasm-bindgen/tree/master/examples/hello_world)就像我们上面看到的那样到[操纵 DOM 节点](https://github.com/alexcrichton/wasm-bindgen/tree/master/examples/dom)完全在 Rust.

在高层次的特点`wasm-bindgen`是:

- 导入 JS 结构,函数,对象等,以调用 wasm. 你可以在一个 struct 和 access 属性上调用 JS 方法,给你写的原生的感觉,一次`#[wasm_bindgen]`所有注释都被连接起来.

- 将 Rust 结构和函数 导出到 JS. 您可以导出 Rust`struct`,而不是让 JS 只使用数字, `struct`会在 JS 中变成了一个`class`. 然后你可以传递结构,而不是只需要传递整数. 该[smorgasboard](https://github.com/alexcrichton/wasm-bindgen/tree/master/examples/smorgasboard)示例, 给出了支持的互操作的良好之味.

- 允许其他各种功能,例如从 全局范围导入 (也称为`alert`函数) ,在 Rust 中使用`Result`捕获 JS 异常,以及模拟在 Rust 程序 中存储 JS 值 的通用方法.

如果您想要看到更多功能,请继续关注[问题跟踪器](https://github.com/alexcrichton/wasm-bindgen/issues)!

## What’s next for `wasm-bindgen`?

在结束之前,我想花点时间描述一下未来的愿景 - `wasm-bindgen`, 因为我认为这是今天该项目最激动人心的方面之一.

### Supporting more than just Rust

从第 1 天开始, `wasm-bindgen`CLI 工具的设计考虑了多种语言支持. 虽然 Rust 是当今唯一受支持的语言, 但该工具也可以插入 C 或 C ++. 该`#[wasm_bindgen]`属性创建`*.wasm`文件输出的自定义部分, 通过`wasm-bindgen`工具解析后,再删除. 本节描述要生成的 JS 绑定 以及 它们的接口. 关于此描述没有特定于 Rust 的内容,因此 C ++ 编译器插件可以轻松地创建并由`wasm-bindgen`工具负责该部分.

我发现这方面特别令人兴奋, 因为我相信它可以实现`wasm-bindgen`工具化成为 WebAssembly 和 JS 集成的标准实践. 希望,它是利于的*所有*语言编译为 WebAssembly 并由 捆绑器 自动识别,以避免上面 几乎所有的配置和构建工具.

### Automatically binding the JS ecosystem

今天导入功能时的缺点之一`#[wasm_bindgen]`宏是, 你必须把所有东西写出来,并确保你没有犯任何错误. 这有时可能是一个繁琐 (且容易出错) 的过程,这个过程已经成熟,可以实现自动化.

所有 Web API 都使用[WebIDL](https://heycam.github.io/webidl/), 它应该是非常可行的[生成`#[wasm_bindgen]`来自 WebIDL 的注释](https://github.com/alexcrichton/wasm-bindgen/issues/42). 这意味着你不需要定义`alert`功能就像我们上面所做的那样,你只需要写下这样的东西:

    #[wasm_bindgen]
    pub fn greet(s: &str) {
        webapi::alert(&format!("Hello, {}!", s));
    }

在这种情况下,`webapi`crate 可以完全从 Web API 的 WebIDL 描述 自动生成,保证没有错误.

我们甚至可以更进一步,利用 TypeScript 社区 的出色工作[生成`#[wasm_bindgen]`也来自 TypeScript](https://github.com/alexcrichton/wasm-bindgen/issues/18). 这将允许任意包 与 npm 上免费提供的 TypeScript 自动绑定!

### Faster-than-JS DOM performance

最后,但并非最不重要的,在`wasm-bindgen`视野: 超快的 DOM 操作 - 许多 JS 框架的核心. 今天,当从 JavaScript 转换到 C ++引擎实现时,调用 DOM 函数 必须经历昂贵的填充程序. 但是,使用 WebAssembly,这些垫片不是必需的. 众所周知,WebAssembly 是一个很好的类型...且,有类型!

该`wasm-bindgen`代码生成是为未来[主机绑定提案](https://github.com/WebAssembly/host-bindings)记录从第 1 天开始而设计的. 只要这是 WebAssembly 中的可用功能,我们将能够直接调用导入的函数,而无需任何 JS 的垫片.此外, `wasm-bindgen`,这将允许 JS 引擎 积极 优化 WebAssembly 操作 DOM, 因为调用是良好类型的,不再需要从 JS 调用 需要的参数验证检查. 在那时候`wasm-bindgen`不仅可以轻松使用 字符串 等更丰富的类型,而且还可以提供 同类最佳 的 DOM 操作 性能.

## Wrapping up

我个人发现 WebAssembly 是令人难以置信的兴奋,不仅因为社区,而且还有如此快速的进步.
`wasm-bindgen`工具前景光明. 它使 JS 和 Rust 之类的语言之间的互操作性成为一流的体验,同时随着 WebAssembly 的不断发展,也提供了长期的好处.

试着给[`wasm-bindgen`](https://github.com/alexcrichton/wasm-bindgen)一个转身机会,[打开一个问题](https://github.com/alexcrichton/wasm-bindgen/issues)用于功能请求,否则[保持参与](http://fitzgeraldnick.com/2018/02/27/wasm-domain-working-group.html)

使用 Rust 和 WebAssembly! 😊❤️

#### Phew

`phew`是感叹词，用于非正式口语中，通常用在当你觉得“解脱”或是“幸好逃过一劫”的时候。 表示松一口气
