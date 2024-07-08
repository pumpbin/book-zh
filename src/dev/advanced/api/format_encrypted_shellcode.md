# format_encrypted_shellcode

将加密后的shellcode转换成另一种格式, 比如隐写到png, 转换成UUID ...

## 函数名

format_encrypted_shellcode

## Input

```json
{
  "shellcode": []
}
```

`shellcode`为字节数组, 是加密后的shellcode.

## Output

```json
{
  "formated_shellcode": []
}
```

`formated_shellcode`为字节数组, 是转换后的shellcode.
