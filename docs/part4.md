# 第四部分 - 我们的第一个测试（和错误）

作者使用的不知道啥脚本，抽空改成Python脚本进行测试





我们不应该允许插入超过列大小的用户名或电子邮件。为了实现这一点，我们需要升级我们的解析器。提醒一下，我们目前使用的是 scanf()：

```c
if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
  statement->type = STATEMENT_INSERT;
  int args_assigned = sscanf(
      input_buffer->buffer, "insert %d %s %s", &(statement->row_to_insert.id),
      statement->row_to_insert.username, statement->row_to_insert.email);
  if (args_assigned < 3) {
    return PREPARE_SYNTAX_ERROR;
  }
  return PREPARE_SUCCESS;
}
```

但是 `scanf` 有一些缺点。如果它正在读取的字符串比它正在读取到的缓冲区大，就会导致缓冲区溢出并开始写入意想不到的地方。我们希望在将每个字符串复制到 ` `Row` ` 结构之前检查其长度。为此，我们需要按空格分割输入。

我将使用 strtok()来实现这一点。我认为如果你看到它实际运行的话，会更容易理解：