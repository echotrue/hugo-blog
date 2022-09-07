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
在大多数数据库中，每当查询被执行时，语句会在后台被预处理。然而，你可以使用`sqlx.DB.Prepare()`显式的预处理语句，以便在其他地方可以重复使用。
```Go
stmt, err := db.Prepare(`SELECT * FROM place WHERE telcode=?`)
row = stmt.QueryRow(65)
 
tx, err := db.Begin()
txStmt, err := tx.Prepare(`SELECT * FROM place WHERE telcode=?`)
row = txStmt.QueryRow(852)
```

`Prepare`实际会在数据库执行预处理操作，所以，它需要占用一个链接。`database/sql`会抽象这一点：通过自动在新的链接上执行预处理操作，允许你使用同一个`Stmt`对象同时在多个连接上执行语句。`Preparex()`返回一个`sqlx.Stmt`对象，它拥有`sqlx.DB`和`sqlx.Tx`两个扩展的所有行为。
```Go
stmt, err := db.Preparex(`SELECT * FROM place WHERE telcode=?`)
var p Place
err = stmt.Get(&p, 852)
```
标准的`sql.Tx`对象也有一个`Stmt()`方法，它从一个已存在的`Stmt`对象中返回一个用于事务的特定`Stmt`对象。`sqlx.Tx`也有一个`Stmtx`的版本，它从一个已存在的`sql.Stmt`或`sqlx.Stmt`对象中创建一个用于事务的特定`sqlx.Stmt`对象。
关于`Stmt`对象，可参考`pkg.go.dev`中`database/sql`文档中的阶段概述：
> Stmt is a prepared statement. A Stmt is safe for concurrent use by multiple goroutines.
> If a Stmt is prepared on a Tx or Conn, it will be bound to a single underlying connection forever. If the Tx or Conn closes, the Stmt will become unusable and all operations will return an error. If a Stmt is prepared on a DB, it will remain usable for the lifetime of the DB. When the Stmt needs to execute on a new underlying connection, it will prepare itself on the new connection automatically.
### Query Helpers
`database/sql`包不对查询语句文本做任何封装操作。这使得在`sql`代码中使用特定于后端的特性变得琐碎。你可以像在数据库中一样迅速的编写查询语句。虽然这很灵活，但是在编写某些类型的查询语句变得困难。

#### In Queries
```Go
var levels = []int{4, 6, 7}
query, args, err := sqlx.In("SELECT * FROM users WHERE level IN (?);", levels)
 
// sqlx.In returns queries with the `?` bindvar, we can rebind it for our backend
query = db.Rebind(query)
rows, err := db.Query(query, args...)
```
`db.Rebind`可以用来获取适用于你的数据库驱动的`query`格式。例如：MySQL使用`?`作为占位符，而SQLite则可以使用`?`和`$1`作为占位符。具体参考`bindvars`章节

#### Named Queries
命名查询，通过映射到结构体字段名或者`map`的`key`来绑定变量到查询。不必映射所有字段。他包含两个与命名查询相关的查询动作：
- NamedQuery(...) (*sqlx.Rows, error) - like Queryx, but with named bindvars
- NamedExec(...) (sql.Result, error) - like Exec, but with named bindvars
和一个额外引用类型查询动作：
- NamedStmt - an sqlx.Stmt which can be prepared with named bindvars

使用示例：
```Go
// named query with a struct
p := Place{Country: "South Africa"}
rows, err := db.NamedQuery(`SELECT * FROM place WHERE country=:country`, p)
 
// named query with a map
m := map[string]interface{}{"city": "Johannesburg"}
result, err := db.NamedExec(`SELECT * FROM place WHERE city=:city`, m)
```
查询所有结果集：
```Go
p := Place{TelephoneCode: 50}
pp := []Place{}
 
// select all telcodes > 50
nstmt, err := db.PrepareNamed(`SELECT * FROM place WHERE telcode > :telcode`)
err = nstmt.Select(&pp, p)

```
命名查询通过解析`:param`语法并将其替换为底层数据库支持的占位符，然后在执行的时候映射查询条件。所以它适用于所有`sqlx`支持的数据库。你也可以使用`sqlx.Named`，他使用`?`占位符，并且可以和`sqlx.In`组合使用。
```Go
arg := map[string]interface{}{
    "published": true,
    "authors": []{8, 19, 32, 44},
}
query, args, err := sqlx.Named("SELECT * FROM articles WHERE published=:published AND author_id IN (:authors)", arg)
query, args, err := sqlx.In(query, args...)
query = db.Rebind(query)
db.Query(query, args...)
```

### Advanced Scanning
`StructScan`看似复杂。他支持结构体嵌套，并且使用与`Go`的属性嵌套及方法访问相同的优先级规则分配字段。一个常见的用法是在多个表之间共享表模型的公共部分。
```Go
type AutoIncr struct {
    ID       uint64
    Created  time.Time
}
 
type Place struct {
    Address string
    AutoIncr
}
 
type Person struct {
    Name string
    AutoIncr
}
```
上面的代码中：`Person`和`Place`将都可以从`StructScan`接收`id`和`created`列的值，因为他们都嵌套了`AutoIncr`结构体。这个特性可以让你快速的为链表查询创建临时表。他可以递归的工作。下面的`Employee`结构体拥有`Person`的`Name`字段以及`AutoIncr`的 `ID`和`Created`字段的访问权限。
```Go
type Employee struct {
    BossID uint64
    EmployeeID uint64
    Person
}
```
注意：`sqlx`历史版本为非嵌入式结构体支持此特性，这使得开发者感到困惑。因为有用户利用此特性定义关系并嵌入相同的结构体两次：
```Go
type Child struct {
    Father Person
    Mother Person
}
```
这会引起一些问题。在Go中隐藏派生字段是合法的.如果上面的`Employee`定义了`Name`字段，他的优先级将会高于`Person`结构体的`Name`字段。但是模糊的选择器是非法的且会引起运行时错误。如果我们想要为`Person`和`Place`快速的创建链表查询，我们应该将`id`定义到哪里？是他们两个结构体都嵌入的`AutoIncr`结构体中？这是否会有错误？
由于`sqlx`构建字段名到字段地址的映射方式，当你扫描数据到结构体时，它并不知道在遍历结构体树的过程中是否遇到了同一个字段名两次。所以，与Go不同，`StructScan`将会选择首次遇到的这个字段。由于Go结构体字段是从上到下排序，并且`sqlx`使用广度优先原则便利以维持优先级原则，因此`StructScan`使用最浅层（多层嵌套情况下），且最前面的定义的字段。
例如：在以下结构体中，`StructScan`会将`id`列的值赋给`Persion.AutoIncr.ID`,也可以通过`Persion.ID`来访问。为了避免混淆，建议在你的SQL种使用`AS`关键字为列设置别名。
```Go
type PersonPlace struct {
    Person
    Place
}
```

#### Scan Destination Safety
通常情况下，如果列的值不能映射到目标数据结构的字段上，`StructScan`会返回一个错误。这模仿了Go中对未使用变量的处理方式，但是与标准库编码解码包例如：`encoding/json`不同。因为SQL通常以比解析`JSON`更可控的方式执行，这些错误通常是编码错误，默认情况下会返回错误。
像未使用的变量一样，你忽略的列是对网络和数据库资源的浪费，并且，在没有映射器告诉你一些东西未找到的情况下，尽早的检测出不兼容的映射和结构体标签中的错别字是非常困难的。
尽管如此，在某些情况下需要忽略没有目标结构的列。为此，每一种引用类型都实现了一个`Unsafe`方法，它返回这个引用的拷贝，这份拷贝关闭了安全功能。
Unsafe()方法说明：
```Go
// Unsafe returns a version of DB which will silently succeed to scan when
// columns in the SQL result have no fields in the destination struct.
// sqlx.Stmt and sqlx.Tx which are created from this DB will inherit its
// safety behavior.
func (db *DB) Unsafe() *DB {
	return &DB{DB: db.DB, driverName: db.driverName, unsafe: true, Mapper: db.Mapper}
}
```
Usage：
```Go
var p Person
// err here is not nil because there are no field destinations for columns in `place`
err = db.Get(&p, "SELECT * FROM person, place LIMIT 1;")

// this will NOT return an error, even though place columns have no destination
udb := db.Unsafe()
err = udb.Get(&p, "SELECT * FROM person, place LIMIT 1;")
```

#### Controlling Name Mappint
用作`StructScan`目标的结构体字段必须大写才能被`sqlx`访问。因此，`sqlx`使用`NameMapper`将`strings.ToLower`应用到字段名以映射他们到查询的结果的列。这并不总是可取的，依赖于数据库对象，所以`sqlx`允许通过多种方式自定义映射。
最简单的一种方式就是通过`sqlx.DB.MapperFunc`为引用类型设置映射，改方法接受一个`func(string)string`类型的参数。如果你的库需要特定的映射器，并且你不想污染`sqlx.DB`，你可以创建一个`DB`拷贝用于特定映射场景。
```Go
// if our db schema uses ALLCAPS columns, we can use normal fields
db.MapperFunc(strings.ToUpper)

// suppose a library uses lowercase columns, we can create a copy
copy := sqlx.NewDb(db.DB, db.DriverName())
copy.MapperFunc(strings.ToLower)

```
`sqlx.DB`使用 `sqlx/reflectx`包的 `Mapper` 来实现底层的映射逻辑，并且通过 `sqlx.DB.Mapper`导出当前活跃的映射器。你可以通过直接设置来自定义数据库上的映射器：
```Go
import "github.com/jmoiron/sqlx/reflectx"

// Create a new mapper which will use the struct field tag "json" instead of "db"
db.Mapper = reflectx.NewMapperFunc("json", strings.ToLower)
```

#### Alternate Scan Types
除了使用 `StructScan` 和 `Scan` ,`sqlx`的查询行结果也可以返回切片或 `map`集合：
```Go
rows, err := db.Queryx("SELECT * FROM place")
for rows.Next() {
    // cols is an []interface{} of all of the column results
    cols, err := rows.SliceScan()
}

rows, err := db.Queryx("SELECT * FROM place")
for rows.Next() {
    results := make(map[string]interface{})
    err = rows.MapScan(results)
}
```
`SliceScan`一般使用在你不知道会返回哪些字段的情况下。`MapScan`与之相同，但 `MapScan`将列映射到 `interface{}`类型值上。这里需要重点注意的是， `rows.Columns()`返回的结果不包含完整的名称。例如：
```Go
SELECT a.id, b.id FROM a NATURAL JOIN b
```
将会导致一个 `[]string{"id","id"}`格式的列结果，这破坏了你的map中的一个结果。

### Custom Types
上面的列子都使用了内置类型来扫描或者查询，但是 `database/sql` 提供了接口允许你使用自定义的类型：
- sql.Scanner allows you to use custom types in a Scan()
- driver.Valuer allows you to use custom types in a Query/QueryRow/Exec
这些是标准的接口，使用他们可以确保在 `database/sql` 上提供的服务可以移植到任何库。具体如何使用请参考 [Built In Interfaces](http://jmoiron.net/blog/built-in-interfaces) 这篇文章，或者查看 `sqlx/types` 包，改包实现了一些标准的实用类型。

### The Connection Pool
准备语句和查询的执行都需要连接，`DB`对象会管理一个连接池，以保证它可以进行安全的并发查询。有两种方法可以控制连接池的大小：
- DB.SetMaxIdleConns(n int)
- DB.SetMaxOpenConns(n int)
默认情况下，池子可以无限增长，并且只要池中没有可用连接，就会创建一个连接。你可以使用 `DB.SetMaxOpenConns`设置最大连接数。未使用的连接会被标记为空闲，如果他们不被需要将会被关闭。为避免建立和关闭大量连接，使用 `DB.SetMaxIdleConns` 将最大空闲连接数设置为适合你查询负载的大小是一种明智的选择。
很容易陷入阻塞连接的困境中，为了防止这种情况：
- 确保 `Scan()` 每个row对象
- 确保通过 `Next()`对每个Rows对象进行完整迭代或调用`Close()`
- 确保每一个事务都通过 `Commit()` 或 `Rollback()` 返回连接
如果你忘记这些操作，连接将会一直阻塞直到被 `GC` 回收，并且你的数据库会立刻停止建立更多的连接，以此抵消当前正在使用的这个链接。注意 `Rows.Close()` 可以被多次安全的调用，所以不用担心在不需要的地方调用它。