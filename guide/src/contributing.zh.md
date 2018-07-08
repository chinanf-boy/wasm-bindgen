
# 帮助`wasm-bindgen`

本节包含有关如何 启动和运行 此项目以进行开发的说明. 

## 先决条件

1.  Rust Nightly. [install Rust]. 安装 Rust 后,运行

    ```shell
    rustup default nightly
    rustup target add wasm32-unknown-unknown
    ```

[install rust]: https://www.rust-lang.org/en-US/install.html

2.  该项目的测试使用 Node. 确保安装了 **Node>=8**, 因为这是在引入 WebAssembly支持 时. [isntall node]. 

[install node]: https://nodejs.org/en/

3.  该项目的测试也使用`yarn` - Node的包管理器. 安装`yarn`, 运行: 

    ```shell
    npm install -g yarn
    ```

    或 遵循其他特定于平台的说明[这里](https://yarnpkg.com/en/docs/install). 

    一旦`yarn`安装后,在顶级目录中运行它: 

    ```shell
    yarn
    ```

## 运行测试

最后,您可以运行测试`cargo`: 

```shell
cargo test
```
