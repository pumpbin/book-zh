# Quick Start

在这一章, 我们将制作一个`Local`类型的Plugin, 并理解PumpBin的工作原理.

`Local`类型的Plugin, shellcode在最终植入物内部存放. 这种方式有一定的稳定性优势, 但是shellcode容易被提取分析, 从而导致内存扫描无法通过.

PumpBin将读取shellcode文件, 并根据加密设置将加密的shellcode动态patch到二进制植入物模板内, 加密所使用的随机密码也会一同patch.

所以PumpBin本质上是一个二进制数据搜索替换工具, 这种实现方式要求预先在二进制植入物内放置占位数据, 一旦编译完成, 占位数据长度将不可更改.

一般情况下, 我们会让shellcode占位数据长度比所需要的稍大一些(如果你知道将要使用的shellcode长度), 这有两个原因:

1. 为了更大的兼容性(如果Plugin shellcode占位数据长度小于加密后的shellcode长度, Plugin将无法用于此shellcode)
1. 多出的占位数据并非无用, PumpBin将用随机数据填充, 保证每次生成的植入物都是唯一的

## 制作二进制植入物模板

我将使用[rust-shellcode](https://github.com/b1nhack/rust-shellcode)中的[create_thread](https://github.com/b1nhack/rust-shellcode/blob/main/create_thread/src/main.rs)作为加载代码.

克隆`rust-shellcode`

```sh
git clone --depth 1 https://github.com/b1nhack/rust-shellcode.git
```

进入`create_thread`目录

```sh
cd rust-shellcode
cd create_thread
```

编译运行, 测试是否成功加载演示shellcode\
默认会运行[w64-exec-calc-shellcode-func.bin](https://github.com/peterferrie/win-exec-calc-shellcode/blob/master/build/bin/w64-exec-calc-shellcode-func.bin), 你应该看到`calc`程序被启动.

```sh
cargo r
```

我们需要使用shellcode占位数据替换`w64-exec-calc-shellcode-func`\
在`create_thread`目录下创建build.rs, 并粘贴以下代码

```rust
use std::{fs, iter};

fn main() {
    let mut shellcode = "$$SHELLCODE$$".as_bytes().to_vec();
    shellcode.extend(iter::repeat(b'0').take(1024 * 1024));
    fs::write("shellcode", shellcode.as_slice()).unwrap();
}
```

这样我们会生成一个大约1MiB的占位数据, 并且由`$$SHELLCODE$$`开头(我们称其为`Prefix`, PumpBin使用它来定位占位数据, 所以`Prefix`需要保持唯一, PumpBin会使用第一个匹配结果).

将main.rs第11行替换为下面的代码, 使其包含占位数据

```rust
let shellcode = include_bytes!("../shellcode");
```

由于shellcode后会填充随机数据, 我们需要知道shellcode的长度, 以正确取出shellcode.

在main.rs 11行代码后添加以下代码

```rust
const SIZE_HOLDER: &str = "$$99999$$";
let shellcode_len = usize::from_str_radix(SIZE_HOLDER, 10).unwrap();
let shellcode = &shellcode[0..shellcode_len];
```

我们添加了一个常量字符串引用, `$$99999$$`也是一个`Prefix`, 不过我们更喜欢称其为`Place Holder`, 因为它将被完全替换.
(`Prefix`会以有效数据+随机数据的形式完全替换, 而`Place Holder`会以有效数据完全替换)

有了shellcode的长度信息, 我们可以获取到正确的shellcode, 编译修改后的`create_thread`项目, 我们将得到一个简单的二进制植入物模板

```sh
cargo b -r
```

## 制作Plugin

现在我们有了一个简单的二进制植入物模板, 我们可以使用PumpBin Maker制作一个只包含Windows Exe的Plugin

Plugin Name填写first_plugin. (这个字段是Plugin的唯一标识, 这意味着用户无法同时安装两个同名的Plugin)

Prefix填写`$$SHELLCODE$$`. (这是我们上面shellcode占位数据的`Prefix`, 你可以使用任何你喜欢的`Prefix`, 只要确保它是唯一的, 或者是第一个被匹配的)

MaxLen填写shellcode占位数据的总大小, 在这个例子中应该是 1024\*1024 + Prefix的大小 = 1048589. (单位是Bytes)

Type保持选中`Local`, Size Holder填写`$$99999$$`. (这是我们上面用来标识shellcode长度的常量字符串引用, 你可以使用任何你喜欢的Size Holder, 规则同上)

Encrypt Type保持None

Windows Exe选择我们上面编译好的二进制植入物模板. (也可以直接填写文件路径)

点击Generate, 保存生成的Plugin

## 测试Plugin

用PumpBin安装制作的Plugin, 并使用`w64-exec-calc-shellcode-func`生成一个最终植入物, 运行应该看到`calc`程序被启动.

至此, 我们制作了一个`Local`类型的Plugin, 并理解了PumpBin的工作原理.

下一章将介绍PumpBin中的加密系统, 你将不必再关心加密过程, 只需要知道每当你在Plugin列表中选择一个Plugin时, PumpBin将为你重新生成对应Plugin的随机加密密码.
(如果你想要重新生成当前选中的Plugin的加密密码, 只需要再次点击Plugin列表中的当前选中插件)

本例中的完整项目文件在PumpBin代码仓库的[examples/create_thread](https://github.com/pumpbin/pumpbin/blob/main/examples/create_thread/src/main.rs).
