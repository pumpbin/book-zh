# encrypt_shellcode

将原始shellcode加密.

## 函数名

encrypt_shellcode

## Input

```json
{
  "shellcode": []
}
```

`shellcode`为字节数组, 是原始shellcode.

## Output

```json
{
  "encrypted": [],
  "pass": [
    {
      "holder": [],
      "replace_by": []
    },
    {
      "holder": [],
      "replace_by": []
    }
  ]
}
```

`encrypted`为字节数组, 是加密后的shellcode.

`pass`为以下json结构数组, 用于patch二进制植入物模板中的加密密码(有的加密方式有多个密码, 所以是一个数组):

```json
{
    "holder": [],
    "replace_by": []
}
```

`holder`为字节数组, 是二进制植入物模板解密函数中的密码占位符.
分享Plug-in时请告知如何正确设置解密函数中的密码.
比如[plug-in](https://github.com/pumpbin/plug-in)仓库中的[aes256-gcm](https://github.com/pumpbin/plug-in/tree/main/encrypt_shellcode/aes256-gcm)有如下READEME内容:

> key: `$$KKKKKKKKKKKKKKKKKKKKKKKKKKKK$$`\
> nonce: `$$NNNNNNNN$$`

上述内容表明使用`aes256-gcm` Plug-in, 需要将二进制植入物模板AES256-GCM解密函数中的key设置为`$$KKKKKKKKKKKKKKKKKKKKKKKKKKKK$$`, nonce设置为`$$NNNNNNNN$$`.

`replace_by`为字节数组, 是加密密码. 可以在Plug-in中生成随机加密密码, 使每个生成的最终植入物都有唯一的加密密码.
