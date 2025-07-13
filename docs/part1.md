# 第一部分 - 简介和设置 REPL

## Sqlite

一个查询需要通过一系列组件来检索或修改数据。前端包括：

- tokenizer 词法分析器

- parser 解析器
- code generator 代码生成器

前端的输入是一个 SQL 查询。输出是 sqlite 虚拟机字节码（本质上是一个可以在数据库上运行的编译程序）。

后端由以下部分组成：

- virtual machine 虚拟机
- B-tree B 树
- pager
- os interface 操作系统接口

虚拟机将前端生成的字节码作为指令。它可以对一个或多个表或索引执行操作，每个表或索引都存储在一个称为 B 树的 数据结构中。虚拟机本质上是一个根据字节码指令类型的大 switch 语句。



每个 B 树由许多节点组成。每个节点是一个页面的长度。B 树可以通过向 pager 发出命令来从磁盘检索页面或将其保存回磁盘。

翻译成简体中文： 分页器接收读取或写入数据页面的命令。它负责在数据库文件中的适当偏移量进行读取/写入。它还保留最近访问的页面在内存中的缓存，并确定这些页面何时需要写回磁盘。

操作系统接口是取决于 sqlite 编译用于哪个操作系统的层。在这个教程中，我不会支持多个平台。



## Making a Simple REPL 制作一个简单的 REPL

Sqlite 在命令行启动时会启动一个读-执行-打印循环：

```sqlite
~ sqlite3
SQLite version 3.16.0 2016-11-04 19:09:39
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
sqlite> create table users (id int, username varchar(255), email varchar(255));
sqlite> .tables
users
sqlite> .exit
~
```

为此，我们的主函数将有一个无限循环，该循环打印提示符，获取一行输入，然后处理该行输入：

```c
int main(int argc, char* argv[]) {
  InputBuffer* input_buffer = new_input_buffer();
  while (true) {
    print_prompt();
    read_input(input_buffer);

    if (strcmp(input_buffer->buffer, ".exit") == 0) {
      close_input_buffer(input_buffer);
      exit(EXIT_SUCCESS);
    } else {
      printf("Unrecognized command '%s'.\n", input_buffer->buffer);
    }
  }
}
```

我们将 `InputBuffer` 定义为一个包装器，用于围绕我们需要存储以与 getline() 交互的状态。（稍后会有更多关于这方面的内容）

```c
typedef struct {
  char* buffer;
  size_t buffer_length;
  ssize_t input_length;
} InputBuffer;

InputBuffer* new_input_buffer() {
  InputBuffer* input_buffer = (InputBuffer*)malloc(sizeof(InputBuffer));
  input_buffer->buffer = NULL;
  input_buffer->buffer_length = 0;
  input_buffer->input_length = 0;

  return input_buffer;
}
```

接下来， `print_prompt()` 向用户打印一个提示符。我们在读取每一行输入之前都会这样做。

```c
void print_prompt() { printf("db > "); }
```

读取一行输入，使用 getline()：

```c
ssize_t getline(char **lineptr, size_t *n, FILE *stream);
```

`lineptr` : 指向我们用来指向包含读取行的缓冲区的变量的指针。如果它被设置为 `NULL` ，则由 `getline` 分配，因此用户应该释放它，即使命令失败。

- `n` : 指向我们用来保存分配缓冲区大小的变量的指针。

- `stream` : 要从中读取的输入流。我们将从标准输入读取

- `return value` : 读取的字节数，可能小于缓冲区的大小

我们告诉 `getline` 将读取的行存储在 `input_buffer->buffer` 中，并将分配的缓冲区大小存储在 `input_buffer->buffer_length` 中。我们将返回值存储在 `input_buffer->input_length` 中。

`buffer` 初始为空，所以 `getline` 分配足够的内存来存储输入的行，并将 `buffer` 指向它。

```c
void read_input(InputBuffer* input_buffer) {
  ssize_t bytes_read =
      getline(&(input_buffer->buffer), &(input_buffer->buffer_length), stdin);

  if (bytes_read <= 0) {
    printf("Error reading input\n");
    exit(EXIT_FAILURE);
  }

  // Ignore trailing newline
  input_buffer->input_length = bytes_read - 1;
  input_buffer->buffer[bytes_read - 1] = 0;
}
```

现在可以定义一个函数来释放为 `InputBuffer *` 实例和相应结构中的 `buffer` 元素（ `getline` 在 `read_input` 中为 `input_buffer->buffer` 分配内存）分配的内存。

```c
void close_input_buffer(InputBuffer* input_buffer) {
    free(input_buffer->buffer);
    free(input_buffer);
}
```

最后，我们解析并执行命令。目前只识别一个命令： `.exit` ，它将终止程序。否则，我们打印错误消息并继续循环。

```c
if (strcmp(input_buffer->buffer, ".exit") == 0) {
  close_input_buffer(input_buffer);
  exit(EXIT_SUCCESS);
} else {
  printf("Unrecognized command '%s'.\n", input_buffer->buffer);
}
```

