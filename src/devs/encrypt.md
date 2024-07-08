# Encryption

在这一章, 我们将优化上一章制作的Plugin, 使用AES256-GCM加密shellcode.

大部分黑客应该都想加密自己的shellcode, 没有人会愿意暴露基础设施.

## 制作二进制植入物模板

要制作有加密功能的Plugin, 我们的二进制植入物模板需要先实现对应的解密逻辑.

我们将在[上一章代码](https://github.com/pumpbin/pumpbin/tree/main/examples/create_thread)的基础上更改.

在Cargo.toml文件末尾添加下面的依赖

```toml
aes-gcm = "0.10.3"
```

在main.rs中的main函数上面添加如下解密函数

```rust
fn decrypt(data: &[u8]) -> Vec<u8> {
    const KEY: &[u8; 32] = b"$$KKKKKKKKKKKKKKKKKKKKKKKKKKKK$$";
    const NONCE: &[u8; 12] = b"$$NNNNNNNN$$";

    let aes = Aes256Gcm::new_from_slice(KEY).unwrap();
    let nonce = Nonce::from_slice(NONCE);
    aes.decrypt(nonce, data).unwrap()
}
```

其中两个被`$$`包裹的数组引用, 上一章已经出现过, 是两个`Place Holder`, PumpBin使用它来定位占位数据.
(`Place Holder`是固定大小, `Prefix`是动态大小, 所以上一章中需要Size Holder来确定shellcode真实长度, 并且还需要Max Len字段)

在main.rs中main函数内第4行后添加如下代码

```rust
let shellcode = decrypt(shellcode);
```

添加完成后的main函数如下

```rust
fn main() {
    let shellcode = include_bytes!("../shellcode");
    const SIZE_HOLDER: &str = "$$99999$$";
    let shellcode_len = usize::from_str_radix(SIZE_HOLDER, 10).unwrap();
    let shellcode = &shellcode[0..shellcode_len];
    let shellcode = decrypt(shellcode);
    let shellcode_size = shellcode.len();
    ...
```

编译修改后的`create_thread`项目, 我们将得到一个使用AES256-GCM解密shellcode的二进制植入物模板.

```sh
cargo b -r
```

## 制作Plugin

我们使用PumpBin Maker制作Plugin, 步骤和上一章相同, 唯一不同的是我们还需要一个实现了对应解密逻辑的Extism Plug-in.

[plug-in](https://github.com/pumpbin/plug-in)仓库收集可重复使用的PumpBin Extism Plug-in, 其中包含[aes256-gcm](https://github.com/pumpbin/plug-in/tree/main/encrypt_shellcode/aes256-gcm) Extism Plug-in.

README.md中说明了使用方法

> key: `$$KKKKKKKKKKKKKKKKKKKKKKKKKKKK$$`\
> nonce: `$$NNNNNNNN$$`

我们需要在二进制植入物模板的解密函数中使用特定的key和nonce作为`Place Holder`, 我们前面已经这样做了.
(如果你前面修改了解密函数中的KEY或者NONCE, 那么你需要从添加解密函数步骤重新开始)

下载编译后的[aes256_gcm.wasm](https://github.com/pumpbin/plug-in/releases), 并将文件路径输入到到PumpBin Maker中的`Encrypt Shellcode Plug-in`输入框中. (也可以使用文件选择对话框选择下载的`aes256_gcm.wasm`文件)

> Plugin Name填写first_plugin. (这个字段是Plugin的唯一标识, 这意味着用户无法同时安装两个同名的Plugin)

> Prefix填写`$$SHELLCODE$$`. (这是我们上面shellcode占位数据的`Prefix`, 你可以使用任何你喜欢的`Prefix`, 只要确保它是唯一的, 或者是第一个被匹配的)

> Max Len填写shellcode占位数据的总大小, 在这个例子中应该是 1024\*1024 + Prefix的大小 = 1048589. (单位是Bytes)

> Type选择`Local`, Size Holder填写`$$99999$$`. (这是我们上面用来确定shellcode长度的常量字符串引用, 你可以使用任何你喜欢的Size Holder, 规则同上)

> Windows Exe选择我们上面编译好的二进制植入物模板. (也可以直接填写文件路径)

> 点击Generate, 保存生成的b1n文件

## 测试Plugin

用PumpBin安装制作的Plugin, 并使用`w64-exec-calc-shellcode-func`生成一个最终植入物, 运行应该看到calc程序被启动.

至此, 我们制作了一个使用AES256-GCM加密方法的`Local`类型Plugin

前两章中, 我总是将`Local`引用起来, 以提醒这是一个关键词, 正确理解它们对于使用PumpBin非常重要.

下一章, 我们将制作第一个`Remote`类型的Plugin. 这允许将shellcode托管在远程服务器.

本例中的完整项目文件在PumpBin代码仓库的[examples/create_thread_encrypt](https://github.com/pumpbin/pumpbin/blob/main/examples/create_thread_encrypt/src/main.rs).
