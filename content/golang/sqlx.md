---
title: "Sqlx"
date: 2022年8月4日14:50:30
draft: false
---

### Handle Types (引用类型)

`sqlx`旨在和`database/sql`具有相同的感觉，他有四种引用类型

- `sqlx.DB` 类似于`sql.DB`,是数据库的表示
- `sqlx.Tx`类似于`sql.Tx`,是事务的表示
- `sqlx.Stmt`类似于`sql.Stmt`,是预处理语句的表示
- `sqlx.NamedStmt`是一种支持命名参数的预处理语句的表示

引用类型都嵌入了他们在`database/sql`中的等效物，这就意味着当你调用`sqlx.DB.Query()`，实际是调用与`sql.DB.Query`相同的代码。

除此之外，还有两种光标类型：



### Querying 101 (查询概述)

引用类型实现了以下相同的操作来查询数据库
- `Exec(...) (sql.Result, error)` - unchanged from database/sql
- `Query(...) (*sql.Rows, error)` - unchanged from database/sql
- `QueryRow(...) *sql.Row` - unchanged from database/sql

以下这些是内置操作的扩展
- `MustExec() sql.Result` -- Exec, but panic on error
- `Queryx(...) (*sqlx.Rows, error)` - Query, but return an sqlx.Rows
- `QueryRowx(...) *sqlx.Row` -- QueryRow, but return an sqlx.Row

还有这些新的语义：
- `Get(dest interface{}, ...) error`
- `Select(dest interface{}, ...) error`


#### Exec and MustExec

这两个方法都是用于执行插入，修改以及DDL语句。区别是他们的错误处理机制不一样，`Exec`会将结果和错误返回，由开发者自行处理。而`MustExec`遇到错误会抛出恐慌`panic`

```Go
// Create table
schema := `CREATE TABLE place (country text,city text NULL,telcode integer);`  
result, err := db.Exec(schema)  
if err != nil {  
   log.Fatalln(err)  
}  

// Insert
schema := `INSERT INTO place (country,telcode) VALUES (?,?)`  
result := db.MustExec(schema, "Hong Kong", 852)

//result.LastInsertId()  
//result.RowsAffected()
```

#### Query

#### QueryRow
#### Get and Select

### Transactions (事务)

### Prepared Statement （预处理语句）





