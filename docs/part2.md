# 第二部分 - 全球最简单的 SQL 编译器和虚拟机

我们正在制作 sqlite 的克隆版本。sqlite 的"前端"是一个 SQL 编译器，它解析一个字符串并输出一个称为字节码的内部表示。

这个字节码会被传递给虚拟机，由虚拟机执行。

将事情分成两个步骤这样做有几个优点：

- 降低每个部分的复杂性（例如，虚拟机不必担心语法错误）
- 允许一次性编译常用查询并缓存字节码以提高性能

考虑到这一点，让我们重构我们的 `main` 函数，并在过程中支持两个新的关键字：

```c
 int main(int argc, char* argv[]) {
   InputBuffer* input_buffer = new_input_buffer();
   while (true) {
     print_prompt();
     read_input(input_buffer);

-    if (strcmp(input_buffer->buffer, ".exit") == 0) {
-      exit(EXIT_SUCCESS);
-    } else {
-      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
+    if (input_buffer->buffer[0] == '.') {
+      switch (do_meta_command(input_buffer)) {
+        case (META_COMMAND_SUCCESS):
+          continue;
+        case (META_COMMAND_UNRECOGNIZED_COMMAND):
+          printf("Unrecognized command '%s'\n", input_buffer->buffer);
+          continue;
+      }
     }
+
+    Statement statement;
+    switch (prepare_statement(input_buffer, &statement)) {
+      case (PREPARE_SUCCESS):
+        break;
+      case (PREPARE_UNRECOGNIZED_STATEMENT):
+        printf("Unrecognized keyword at start of '%s'.\n",
+               input_buffer->buffer);
+        continue;
+    }
+
+    execute_statement(&statement);
+    printf("Executed.\n");
   }
 }
```

非 SQL 语句如 `.exit` 被称为"元命令"。它们都以点开头，所以我们检查它们并在一个单独的函数中处理。

接下来，我们添加一个步骤将输入行转换为语句的内部表示。这是我们简陋的 sqlite 前端版本。

最后，我们将准备好的语句传递给 `execute_statement` 。这个函数最终将成为我们的虚拟机。

请注意，我们新增的两个函数返回枚举值表示成功或失败：

```c
typedef enum {
  META_COMMAND_SUCCESS,
  META_COMMAND_UNRECOGNIZED_COMMAND
} MetaCommandResult;

typedef enum { PREPARE_SUCCESS, PREPARE_UNRECOGNIZED_STATEMENT } PrepareResult;
```

"无法识别的语句"？这似乎有点像是一个例外。我不喜欢使用异常（而且 C 语言甚至不支持异常），所以我在所有实用的地方使用枚举结果代码。如果我的 switch 语句没有处理枚举成员，C 编译器会报错，所以我们可以稍微更有信心地处理函数的每个结果。未来可能会添加更多的结果代码。

`do_meta_command` 只是一个现有功能的包装器，为更多命令留有空间：

```c
MetaCommandResult do_meta_command(InputBuffer* input_buffer) {
  if (strcmp(input_buffer->buffer, ".exit") == 0) {
    exit(EXIT_SUCCESS);
  } else {
    return META_COMMAND_UNRECOGNIZED_COMMAND;
  }
}
```

我们当前的"准备语句"只包含一个有两个可能值的枚举。随着我们允许在语句中使用参数，它将包含更多数据：

```c
typedef enum { STATEMENT_INSERT, STATEMENT_SELECT } StatementType;

typedef struct {
  StatementType type;
} Statement;
```

`prepare_statement` （我们的"SQL 编译器"）目前还不理解 SQL。事实上，它只理解两个词：

```c
PrepareResult prepare_statement(InputBuffer* input_buffer,
                                Statement* statement) {
  if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
    statement->type = STATEMENT_INSERT;
    return PREPARE_SUCCESS;
  }
  if (strcmp(input_buffer->buffer, "select") == 0) {
    statement->type = STATEMENT_SELECT;
    return PREPARE_SUCCESS;
  }

  return PREPARE_UNRECOGNIZED_STATEMENT;
}
```

请注意，我们使用 `strncmp` 表示 "insert"，因为 "insert" 关键字后面会跟数据。（例如 `insert 1 cstack foo@bar.com` ）

最后， `execute_statement` 包含几个占位符：

```c
void execute_statement(Statement* statement) {
  switch (statement->type) {
    case (STATEMENT_INSERT):
      printf("This is where we would do an insert.\n");
      break;
    case (STATEMENT_SELECT):
      printf("This is where we would do a select.\n");
      break;
  }
}
```

请注意，它不会返回任何错误代码，因为目前还没有可能出现问题的东西。

通过这些重构，我们现在识别了两个新的关键字！