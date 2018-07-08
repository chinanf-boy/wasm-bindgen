
# CLI参考

该`wasm-bindgen`工具有许多选项可用于调整生成的JS. 默认情况下,生成的JS 使用 ES模块,并且与 Node 和 浏览器兼容 (但可能需要两个用例的捆绑器) . 

可以通过以下方式了解更多`wasm-bindgen --help`, 但一些值得注意的选择是: 

-   `--nodejs`: 这个标志将为输出 Node 而不是 浏览器 ,允许本机使用`require`生成的JS 和
内部使用`require`而不是 ES模块. 使用此标志时,不需要进一步后处理 (也称为捆绑器) 来使用 wasm. 

-   `--browser`: 此标志将专门输出为 浏览器,使其与 Node 不兼容. 这基本上会使生成的JS稍微小一些,因为不需要对 Node 进行运行时检查. 

-   `--no-modules`: `wasm-bindgen`的默认输出是ES模块 ,但此选项表示 不应使用 ES模块,并且应为 Web浏览器定制输出. 在这种模式下`window.wasm_bindgen`将是一个函数,它接受 wasm文件 的 路径 来获取和实例化. 之后,可以通过以下方式从 wasm 导出函数 - `window.wasm_bindgen.foo`. 注意名称`wasm_bindgen`可配置`--no-modules-global FOO`选项. 

-   `--no-typescript`: 默认情况下`*.d.ts`生成,但此标志将禁用生成此TypeScript文件. 

-   `--debug`: 在"调试模式"中生成更多 JS和wasm 以帮助捕获程序员错误,但生成的文件时不打算用于生产环境
