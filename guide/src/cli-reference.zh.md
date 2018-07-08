
# CLI参考

该`wasm-bindgen`工具有许多选项可用于调整生成的JS. 默认情况下,生成的JS使用ES模块,并且与Node和浏览器兼容 (但可能需要两个用例的捆绑器) . 

可以通过以下方式了解CLI工具的支持标志`wasm-bindgen --help`,但一些值得注意的选择是: 

-   `--nodejs`: 这个标志将为Node而不是浏览器定制输出,允许本机使用`require`生成的JS和内部使用`require`而不是ES模块. 使用此标志时,不需要进一步后处理 (也称为捆绑器) 来使用wasm. 

-   `--browser`: 此标志将专门为浏览器定制输出,使其与Node不兼容. 这基本上会使生成的JS稍微小一些,因为不需要对Node进行运行时检查. 

-   `--no-modules`: 默认输出`wasm-bindgen`使用ES模块,但此选项表示不应使用ES模块,并且应为Web浏览器定制输出. 在这种模式下`window.wasm_bindgen`将是一个函数,它接受wasm文件的路径来获取和实例化. 之后,可以通过以下方式从wasm导出函数`window.wasm_bindgen.foo`. 注意名称`wasm_bindgen`可配置`--no-modules-global FOO`旗. 

-   `--no-typescript`: 默认情况下`*.d.ts`为生成的JS文件生成文件,但此标志将禁用生成此TypeScript文件. 

-   `--debug`: 在"调试模式"中生成更多JS和wasm以帮助捕获程序员错误,但此输出不打算发送到生产
