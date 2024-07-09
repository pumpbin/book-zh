# Extism Plug-in

PumpBin使用[Extism](https://github.com/extism/extism)实现插件系统.

> 注意: Extism的主要用例之一是构建可扩展的软件和插件. 您希望能够从用户那里执行任意, 不受信任的代码? Extism使这样做既安全又实用.

PumpBin Extism Plug-in输入输出均是字节流, 并使用json序列化, Plug-in需要将输入字节流反序列化成json, 将json输出序列化成字节流.
在rust中使用[serde_json](https://crates.io/crates/serde_json)的示例如下:

```rust
#[derive(Debug, Serialize, Deserialize)]
pub struct Input {
    ...
}

#[derive(Debug, Serialize, Deserialize)]
pub struct Output {
    ...
}

#[plugin_fn]
pub fn extism_plugin(input: Vec<u8>) -> FnResult<Vec<u8>> {
    let input = serde_json::from_slice::<Input>(input.as_slice())?;
    ...
    let output = Output { ... };
    Ok(serde_json::to_vec(&output)?)
}
```

<div class="warning">

常见错误

如果Plug-in中有任何代码涉及syscalls, 则需要编译为启用`wasi`模块. 比如生成随机数, 网络访问等等. (推荐直接编译为启用`wasi`模块, 避免错误, rust中为`wasm32-wasi` target)

</div>

Extism官网有使用各种语言开发Plug-in的[详细教程](https://extism.org/docs/quickstart/plugin-quickstart), 下面是PumpBin Extism Plug-in的API文档.

- [encrypt_shellcode](dev/advanced/api/encrypt_shellcode.md)
- [format_encrypted_shellcode](dev/advanced/api/format_encrypted_shellcode.md)
- [format_url_remote](dev/advanced/api/format_url_remote.md)
- [upload_final_shellcode_remote](dev/advanced/api/upload_final_shellcode_remote.md)

以remote结尾的是`Remote`类型Plugin特有的Extism Plug-in.
