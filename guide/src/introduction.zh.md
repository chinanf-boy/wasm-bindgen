
# 介绍

`wasm-bindgen`促进 wasm模块和 JavaScript 之间的高级交互. 

这个项目有点像一半 polyfill 的[主机绑定提案][host],和一半用于增强 JS和wasm 编译代码之间的高级交互 (目前主要来自Rust) . 更具体地说,这个项目允许 JS / wasm 通过 字符串,JS对象,类 等进行通信,而不是纯粹的 整数和浮点数. 运用`wasm-bindgen`做某事,比如: 您可以在Rust中定义JS类 或 从JS获取一个字符串 或 返回一个. 这个项目在变强 `{秃的也会很快}`!

目前这个工具以 Rust 为重点,但 底层基础 是与语言无关的,并且希望随着时间的推移,这个工具可以稳定使用,用于 C/C ++ 等语言!

该项目的显着特点包括: 

-   将 JS函数 导入到 Rust 中, 像[DOM操作][dom-ex],[控制台记录][console-log], 要么[性能监控][perf-ex]. 
-   [导出Rust功能][smorg-ex]到 JS,如类,函数等
-   使用丰富的类型,如字符串,数字,类,闭包和对象,而不是简单`u32`和浮点. 

这个项目仍然相对较新,但反馈当然总是受欢迎! 如果你对设计很好奇,加上关于这个箱子可以做什么的更多信息,请查看[desgin doc]. 

[host]: https://github.com/WebAssembly/host-bindings

[design doc]: https://rustwasm.github.io/wasm-bindgen/design.html

[dom-ex]: https://github.com/rustwasm/wasm-bindgen/tree/master/examples/dom

[console-log]: https://github.com/rustwasm/wasm-bindgen/tree/master/examples/console_log

[perf-ex]: https://github.com/rustwasm/wasm-bindgen/tree/master/examples/performance

[smorg-ex]: https://github.com/rustwasm/wasm-bindgen/tree/master/examples/smorgasboard

[hello-online]: https://webassembly.studio/?f=gzubao6tg3
