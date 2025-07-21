#  第六部分 - 游标抽象

这一部分应该比上一部分短。我们只是稍微重构一下，以便更容易开始 B-Tree 的实现。

我们将添加一个 `Cursor` 对象，它代表表中的一个位置。你可以使用游标做的事情：

- Create a cursor at the beginning of the table
  在表的开始处创建一个游标
- Create a cursor at the end of the table
  在表的末尾创建一个游标
- Access the row the cursor is pointing to
  访问游标指向的行
- Advance the cursor to the next row
  将游标移动到下一行

这些是我们现在将要实现的行为。稍后，我们也希望：

- Delete the row pointed to by a cursor
  删除光标指向的行
- Modify the row pointed to by a cursor
  修改光标指向的行
- Search a table for a given ID, and create a cursor pointing to the row with that ID
  在表中搜索给定 ID，并创建一个指向具有该 ID 的行的光标

话不多说，这是 `Cursor` 类型：

```c
+typedef struct {
+  Table* table;
+  uint32_t row_num;
+  bool end_of_table;  // Indicates a position one past the last element
+} Cursor;
```

根据我们当前的表格数据结构，您只需要行号即可识别表格中的位置。光标还包含它所属表格的引用（因此我们的光标函数只需要光标作为参数）。

最后，它有一个名为 `end_of_table` 的布尔值。这是为了我们可以表示表格末尾之后的位置（这是我们可能想要插入行的地方）。

`table_start()` 和 `table_end()` 创建新的光标：

```c
+Cursor* table_start(Table* table) {
+  Cursor* cursor = malloc(sizeof(Cursor));
+  cursor->table = table;
+  cursor->row_num = 0;
+  cursor->end_of_table = (table->num_rows == 0);
+
+  return cursor;
+}
+
+Cursor* table_end(Table* table) {
+  Cursor* cursor = malloc(sizeof(Cursor));
+  cursor->table = table;
+  cursor->row_num = table->num_rows;
+  cursor->end_of_table = true;
+
+  return cursor;
+}
```

我们的 `row_slot()` 函数将变为 `cursor_value()` ，它返回一个指向光标所描述位置的指针：

```c
-void* row_slot(Table* table, uint32_t row_num) {
+void* cursor_value(Cursor* cursor) {
+  uint32_t row_num = cursor->row_num;
   uint32_t page_num = row_num / ROWS_PER_PAGE;
-  void* page = get_page(table->pager, page_num);
+  void* page = get_page(cursor->table->pager, page_num);
   uint32_t row_offset = row_num % ROWS_PER_PAGE;
   uint32_t byte_offset = row_offset * ROW_SIZE;
   return page + byte_offset;
 }
```

在我们的当前表结构中，移动光标很简单，只需增加行号即可。在 B 树中，这会稍微复杂一些。

```c
+void cursor_advance(Cursor* cursor) {
+  cursor->row_num += 1;
+  if (cursor->row_num >= cursor->table->num_rows) {
+    cursor->end_of_table = true;
+  }
+}
```

最后，我们可以将我们的“虚拟机”方法改为使用光标抽象。当插入一行时，我们在表的末尾打开一个光标，写入该光标位置，然后关闭光标。

```c
   Row* row_to_insert = &(statement->row_to_insert);
+  Cursor* cursor = table_end(table);

-  serialize_row(row_to_insert, row_slot(table, table->num_rows));
+  serialize_row(row_to_insert, cursor_value(cursor));
   table->num_rows += 1;

+  free(cursor);
+
   return EXECUTE_SUCCESS;
 }
```

当选择表中的所有行时，我们在表的开始处打开一个光标，打印该行，然后将光标移动到下一行。重复直到到达表的末尾。

```c
 ExecuteResult execute_select(Statement* statement, Table* table) {
+  Cursor* cursor = table_start(table);
+
   Row row;
-  for (uint32_t i = 0; i < table->num_rows; i++) {
-    deserialize_row(row_slot(table, i), &row);
+  while (!(cursor->end_of_table)) {
+    deserialize_row(cursor_value(cursor), &row);
     print_row(&row);
+    cursor_advance(cursor);
   }
+
+  free(cursor);
+
   return EXECUTE_SUCCESS;
 }
```

好的，就这样！正如我所说，这是一个较短的重构，应该能帮助我们重写表数据结构为 B 树。 `execute_select()` 和 `execute_insert()` 可以通过游标完全与表交互，而无需假设表是如何存储的。