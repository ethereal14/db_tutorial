# 第 3 部分 - 一个内存中、仅追加、单表数据库

我们将从小处着手，对数据库施加许多限制。目前，它将：

- support two operations: inserting a row and printing all rows
  支持两种操作：插入行和打印所有行
- reside only in memory (no persistence to disk)
  仅驻留在内存中（不持久化到磁盘）
- support a single, hard-coded table
  支持单个硬编码的表

我们的硬编码表将用于存储用户，看起来像这样：

| column 列 | type 类型    |
| --------- | ------------ |
| id        | integer      |
| username  | varchar(32)  |
| email     | varchar(255) |

这是一个简单的模式，但它让我们能够支持多种数据类型以及多种大小的文本数据类型。

`insert` 语句现在将看起来像这样：

```sqlite
insert 1 cstack foo@bar.com
```

这意味着我们需要升级我们的 `prepare_statement` 函数来解析参数

```c
   if (strncmp(input_buffer->buffer, "insert", 6) == 0) {
     statement->type = STATEMENT_INSERT;
+    int args_assigned = sscanf(
+        input_buffer->buffer, "insert %d %s %s", &(statement->row_to_insert.id),
+        statement->row_to_insert.username, statement->row_to_insert.email);
+    if (args_assigned < 3) {
+      return PREPARE_SYNTAX_ERROR;
+    }
     return PREPARE_SUCCESS;
   }
   if (strcmp(input_buffer->buffer, "select") == 0) {
```

我们将这些解析后的参数存储在语句对象内部的一个新的 `Row` 数据结构中：

```c
+#define COLUMN_USERNAME_SIZE 32
+#define COLUMN_EMAIL_SIZE 255
+typedef struct {
+  uint32_t id;
+  char username[COLUMN_USERNAME_SIZE];
+  char email[COLUMN_EMAIL_SIZE];
+} Row;
+
 typedef struct {
   StatementType type;
+  Row row_to_insert;  // only used by insert statement
 } Statement;
```



现在我们需要将这些数据复制到表示表的数据结构中。SQLite 使用 B 树进行快速查找、插入和删除。我们将从一个更简单的东西开始。像 B 树一样，它将行分组到页面中，但不是以树的形式排列这些页面，而是以数组的形式排列。

Here's my plan: 这是我的计划：

- Store rows in blocks of memory called pages
  将行存储在称为页面的内存块中
- Each page stores as many rows as it can fit
  每页存储尽可能多的行
- Rows are serialized into a compact representation with each page
  行被序列化为每页的紧凑表示
- Pages are only allocated as needed
  页面仅在需要时才分配
- Keep a fixed-size array of pointers to pages
  保留一个固定大小的页面指针数组

首先，我们将定义行的紧凑表示：

```
+#define size_of_attribute(Struct, Attribute) sizeof(((Struct*)0)->Attribute)
+
+const uint32_t ID_SIZE = size_of_attribute(Row, id);
+const uint32_t USERNAME_SIZE = size_of_attribute(Row, username);
+const uint32_t EMAIL_SIZE = size_of_attribute(Row, email);
+const uint32_t ID_OFFSET = 0;
+const uint32_t USERNAME_OFFSET = ID_OFFSET + ID_SIZE;
+const uint32_t EMAIL_OFFSET = USERNAME_OFFSET + USERNAME_SIZE;
+const uint32_t ROW_SIZE = ID_SIZE + USERNAME_SIZE + EMAIL_SIZE;
```

这意味着序列化行的布局将如下所示：

| column 列       | size (bytes) 大小（字节） | offset 偏移量 |
| --------------- | ------------------------- | ------------- |
| id              | 4                         | 0             |
| username 用户名 | 32                        | 4             |
| email           | 255                       | 36            |
| total 总计      | 291                       |               |

我们还需要代码来实现与紧凑表示之间的转换。

```c
+void serialize_row(Row* source, void* destination) {
+  memcpy(destination + ID_OFFSET, &(source->id), ID_SIZE);
+  memcpy(destination + USERNAME_OFFSET, &(source->username), USERNAME_SIZE);
+  memcpy(destination + EMAIL_OFFSET, &(source->email), EMAIL_SIZE);
+}
+
+void deserialize_row(void* source, Row* destination) {
+  memcpy(&(destination->id), source + ID_OFFSET, ID_SIZE);
+  memcpy(&(destination->username), source + USERNAME_OFFSET, USERNAME_SIZE);
+  memcpy(&(destination->email), source + EMAIL_OFFSET, EMAIL_SIZE);
+}
```

接下来，一个 `Table` 结构，它指向包含行的页面并跟踪行数：

```c
+const uint32_t PAGE_SIZE = 4096;
+#define TABLE_MAX_PAGES 100
+const uint32_t ROWS_PER_PAGE = PAGE_SIZE / ROW_SIZE;
+const uint32_t TABLE_MAX_ROWS = ROWS_PER_PAGE * TABLE_MAX_PAGES;
+
+typedef struct {
+  uint32_t num_rows;
+  void* pages[TABLE_MAX_PAGES];
+} Table;
```

我在将我们的页面大小设置为 4 千字节，因为这与大多数计算机架构的虚拟内存系统中的页面大小相同。这意味着我们数据库中的一个页面对应于操作系统使用的一个页面。操作系统将页面作为整体单元在内存中移动，而不是将它们拆分。

我设定了一个任意的限制，即我们将分配 100 页。当我们切换到树结构时，我们数据库的最大大小将仅受文件最大大小的限制。（尽管我们仍然会限制一次在内存中保留的页面数量）

行不应跨越页面边界。由于页面在内存中可能不会相邻存在，这一假设使得读取/写入行更加容易。

说到这个，这里是我们如何确定特定行在内存中的读取/写入位置的方法：

```c
+void* row_slot(Table* table, uint32_t row_num) {
+  uint32_t page_num = row_num / ROWS_PER_PAGE;
+  void* page = table->pages[page_num];
+  if (page == NULL) {
+    // Allocate memory only when we try to access page
+    page = table->pages[page_num] = malloc(PAGE_SIZE);
+  }
+  uint32_t row_offset = row_num % ROWS_PER_PAGE;
+  uint32_t byte_offset = row_offset * ROW_SIZE;
+  return page + byte_offset;
+}
```

现在我们可以让 `execute_statement` 从我们的表结构中读写：

```c
-void execute_statement(Statement* statement) {
+ExecuteResult execute_insert(Statement* statement, Table* table) {
+  if (table->num_rows >= TABLE_MAX_ROWS) {
+    return EXECUTE_TABLE_FULL;
+  }
+
+  Row* row_to_insert = &(statement->row_to_insert);
+
+  serialize_row(row_to_insert, row_slot(table, table->num_rows));
+  table->num_rows += 1;
+
+  return EXECUTE_SUCCESS;
+}
+
+ExecuteResult execute_select(Statement* statement, Table* table) {
+  Row row;
+  for (uint32_t i = 0; i < table->num_rows; i++) {
+    deserialize_row(row_slot(table, i), &row);
+    print_row(&row);
+  }
+  return EXECUTE_SUCCESS;
+}
+
+ExecuteResult execute_statement(Statement* statement, Table* table) {
   switch (statement->type) {
     case (STATEMENT_INSERT):
-      printf("This is where we would do an insert.\n");
-      break;
+      return execute_insert(statement, table);
     case (STATEMENT_SELECT):
-      printf("This is where we would do a select.\n");
-      break;
+      return execute_select(statement, table);
   }
 }
```

