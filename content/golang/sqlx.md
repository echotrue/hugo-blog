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
```Go
// fetch all places from the db
rows, err := db.Query("SELECT country, city, telcode FROM place")

// iterate over each row
for rows.Next() {
	var country string
	// note that city can be NULL, so we use the NullString type
	var city sql.NullString
	var telcode int
	err = rows.Scan(&country, &city, &telcode)
}
// check the error from rows
err = rows.Err()
```

应该像数据库游标一样看待`Rows`，而不是将其看作一个具体的结果集。尽管数据库驱动缓冲行为有所不同，但是通过`Next()`迭代能够有效的约束大型查询结果集的内存使用，因为一次只扫描一行。`Scan()`通过反射来将数据库列的类型映射到`Go`的类型，例如：string,[]byte等等。如果不迭代所有行的结果，请确保调用`rows.Close()`以便将连接返回到连接池。
`Query`查询返回的`error`是数据库服务`preparing`或`executing`过程中发生的任意错误。这些错误包含从连接池中获取无效的连接，尽管`database/sql`会重试10次来尝试找到或创建正常的连接。通常，这些错误是由SQL语法错误，类型错误，字段名和表明错误引起。
大多数情况下，`Rows.Scan`会复制其从数据库驱动中获取的数据，因为它不知道数据库驱动将会如何重复使用缓冲区。特殊类型`sql.RawBytes`可以用来从数据库驱动返回的实际数据中获取一个零拷贝的字节切片。下次调用`Next()`之后，这个值将不再有效，因为这个值所在的内存可能已经被覆盖。
`Query`查询使用的连接会一直保持活跃状态，直到通过`Next()`迭代完查询出的所有行，或者`rows.Close()`被调用。此时，连接才会被释放。
`Queryx`和`Query`用法基本一样，只是他返回一个`sqlx.Rows`对象，这个对象拥有更加丰富的`scan`行为可供选择。

```
type Place struct {
    Country       string
    City          sql.NullString
    TelephoneCode int `db:"telcode"`
}
 
rows, err := db.Queryx("SELECT * FROM place")
for rows.Next() {
    var p Place
    err = rows.StructScan(&p)
}
```
`StructScan()`是`sql.Rows`主要的扩展行为，他会扫描查询结果并映射到结构体字段中。注意，这些字段必须是导出的（首字母大写），以便`sqlx`能够将数据写入，这个规则适用于Go中的所有类型（json,xml）的`Marshalling`(编码)和`UNMarshalling`(解码)。可以使用结构体标签`db`指定数据表列名和结构体字段的映射关系，或者使用`db.MapperFunc()`设置新的默认映射关系。默认是使用`strings.Lower`将结构体字段转小写来匹配数据表列名。

#### QueryRow
`QueryRow`从数据表提取一条记录。它从连接池中获取一个连接，然后通过`Query`执行查询，返回一个内部包含`Rows`对象的`Row`对象。
与`Query`不同，`QueryRow`返回没有错误的`Row`类型结果，使得对查询结果的链式`Scan`操作是安全的。如果执行查询过程中发生错误，错误会通过`Scan`返回。如果没有查询到结果，`Scan`返回`sql.ErrNoRows`错误。如果`Scan`本身发生错误（例如：类型不匹配）,错误同样会被返回。
查询结果`Row`内部的`Rows`结构体在`Scan`执行完后会关闭，这就意味着，`QueryRow`查询所在的数据库连接一直会保持到查询结果被扫描完。同样意味着`sql.RawBytes`在这里也不可用，由于被引用的内存块属于数据库驱动，并且调用返回时该内存块可能已经无效。
`sqlx`扩展的方法`QueryRowx`会返回`sqlx.Row`代替`sql.Row`，它实现了上述介绍以及高阶扫描中`Rows`相同的扫描扩展。
#### Get and Select
`Get`和`Select`是引用类型的节省时间的扩展，它将查询的执行和灵活的扫描语义相结合。为了清楚的解释它们，我们不得不谈论下什么是`scannable`(可扫描的)：
- 不是结构体类型的值是可扫描的，例如：`string`, `int`
- 实现了`sql.Scanner`接口的值是可扫描的
- 没有导出字段的结构体可以是扫描的(eg. `time.Time`)
`Get`和`Select`使用`rows.Scan`在可扫描类型上，使用`rows.StructScan`在不可扫描类型上。它们与`QueryRow`和`Query`大致相似，`Get`主要用来获取单条记录并扫描它，`Select`主要用来获取一个结果集。
```Go
p := Place{}
pp := []Place{}
 
// this will pull the first place directly into p
err = db.Get(&p, "SELECT * FROM place LIMIT 1")
 
// this will pull places with telcode > 50 into the slice pp
err = db.Select(&pp, "SELECT * FROM place WHERE telcode > ?", 50)
 
// they work with regular types as well
var id int
err = db.Get(&id, "SELECT count(*) FROM place")
 
// fetch at most 10 place names
var names []string
err = db.Select(&names, "SELECT name FROM place LIMIT 10")
```
`Get`和`Select`都将会关闭它们在查询执行过程中创建的`Rows`对象，并且返回在这个过程中发生的任何错误。由于它们在内部使用`StructScan`,因此"高级扫描"中介绍的部分也适用于这两个方法。
`Select`可以为你节省很多输入，但是，请谨记！它在语义上和`Queryx`不同，因为它会将整个查询结果集一次性的加载进内存，如果这个结果集没有被查询限制到一个合理的大小，使用经典的`Queryx/StructScan`迭代反而是更好的选择。

### Transactions (事务)
要使用事务，首选需要通过`DB.Begin()`创建一个事务的引用对象。记住，`Exec`以及其他查询动作每次都会向`DB`索要一个连接，并且最终会将连接放回连接池。由于无法保证你每次索要的连接与`Begin()`执行时所在的连接是同一个连接，所以，要使用事务，必须先调用`DB.Begin()`.正确的使用方法如下：
```Go
tx, err := db.Begin()
err = tx.Exec(...)
err = tx.Commit()
```
`DB`引用也有扩展行为`Beginx()`和`MustBegin()`,他们返回`sqlx.Tx`而不是`sql.Tx`。`sqlx.Tx`拥有`sqlx.DB`的所有扩展行为。
一旦事务是连接状态，`Tx`对象必须绑定并限定为单个从池中获取的连接，`Tx`在其整个生命周期中都将维持单个连接，只有调用`Commit()`或`Rollback()`才会释放连接。需要注意的是你应该至少调用这两个方法中的一个，否则，连接将会一直保持直到被GC回收。
由于在事务中只有一个连接可以被使用，所以一次只能执行一条语句。在执行其他查询之前，必须分别扫描完或关闭`Row` 和 `Rows`。在数据库服务器向你发送结果的时候，如果你尝试向数据库发送数据，这很可能会破坏当前连接。
`Tx`对象实际并不意味着在服务器上的任何行为，它只是执行了`begin`语句并绑定了单个连接。事务的实际行为，诸如：锁定和隔离等，在此不做具体说明，这些依赖于数据库。

### Prepared Statement （预处理语句）
在大多数数据库中，每当查询被执行时，语句会在后台被预处理。然后，你可以使用`sqlx.DB.Prepare()`显式的预处理语句，以便在其他地方可以重复使用。
```Go
stmt, err := db.Prepare(`SELECT * FROM place WHERE telcode=?`)
row = stmt.QueryRow(65)
 
tx, err := db.Begin()
txStmt, err := tx.Prepare(`SELECT * FROM place WHERE telcode=?`)
row = txStmt.QueryRow(852)
```






