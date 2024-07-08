# upload_final_shellcode_remote

将final shellcode上传到远程服务器. (`Remote` Only)

final shellcode是经过加密和转换后的shellcode, 如果某个Plug-in未设置, 则原样返回, 例如format_encrypted_shellcode Plug-in未设置, 则原样返回传入的加密后的shellcode.

## 函数名

upload_final_shellcode_remote

## Input

```json
{
  "final_shellcode": []
}
```

`final_shellcode`为字节数组, 是final shellcode.

## Output

```json
{
  "url": ""
}
```

`url`为字符串, 是上传后的url地址, PumpBin将自动填写到shellcode url输入框中.
