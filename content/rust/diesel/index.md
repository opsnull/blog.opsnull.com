---
title: "diesel"
author: ["zhangjun"]
lastmod: 2024-08-02T20:20:58+08:00
draft: false
series: ["rust crate"]
series_order: 18
---

## <span class="section-num">1</span> 安装 diesel_cli {#安装-diesel-cli}

```shell
brew install mysql-client sqlite postgresql

zj@a:~/work/code/learn-by-doing$ ls -l /opt/homebrew/opt/mysql-client/lib
total 14M
-rw-r--r-- 1 alizj 6.6M  8  1 13:44 libmysqlclient.23.dylib
-r--r--r-- 1 alizj 7.1M 12 14  2023 libmysqlclient.a
lrwxr-xr-x 1 alizj   23 12 14  2023 libmysqlclient.dylib -> libmysqlclient.23.dylib
drwxr-xr-x 3 alizj   96  8  1 13:44 pkgconfig/

export MYSQLCLIENT_LIB_DIR=/opt/homebrew/opt/mysql-client/lib MYSQLCLIENT_VERSION=23
cargo install diesel_cli

# 启动 postgresql
brew services start postgresql@14

zj@a:~/work/code/learn-by-doing/rust/diesel_demo$ brew services list
Name          Status  User  File
bind          none
dbus          none
emacs-plus@30 none
postgresql@14 started alizj ~/Library/LaunchAgents/homebrew.mxcl.postgresql@14.plist
unbound       none

zj@a:~/work/code/learn-by-doing/rust/diesel_demo$ lsof -i tcp:5432
COMMAND    PID  USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
postgres 64220 alizj    7u  IPv6 0x5297ba1d8b4e0e13      0t0  TCP localhost:postgresql (LISTEN)
postgres 64220 alizj    8u  IPv4 0x5297ba271f55c663      0t0  TCP localhost:postgresql (LISTEN)

zj@a:~/work/code/learn-by-doing/rust/diesel_demo$ psql -d postgres
psql (14.12 (Homebrew))
Type "help" for help.

postgres=# help
You are using psql, the command-line interface to PostgreSQL.
Type:  \copyright for distribution terms
       \h for help with SQL commands
       \? for help with psql commands
       \g or terminate with semicolon to execute query
       \q to quit
postgres=# CREATE ROLE zj WITH LOGIN PASSWORD '1234';
CREATE ROLE
postgres=# ALTER ROLE zj CREATEDB;
ALTER ROLE

zj@a:~/work/code/learn-by-doing/rust/diesel_demo$ psql -l
                         List of databases
   Name    | Owner | Encoding | Collate | Ctype | Access privileges
-----------+-------+----------+---------+-------+-------------------
 postgres  | alizj | UTF8     | C       | C     |
 template0 | alizj | UTF8     | C       | C     | =c/alizj         +
           |       |          |         |       | alizj=CTc/alizj
 template1 | alizj | UTF8     | C       | C     | =c/alizj         +
           |       |          |         |       | alizj=CTc/alizj
(3 rows)

zj@a:~/work/code/learn-by-doing/rust/diesel_demo$ echo DATABASE_URL=postgres://zj:1234@localhost/diesel_demo > .env

zj@a:~/work/code/learn-by-doing/rust/diesel_demo$ diesel setup
Creating migrations directory at: /Users/alizj/work/code/learn-by-doing/rust/diesel_demo/migrations
Creating database: diesel_demo

zj@a:~/work/code/learn-by-doing/rust/diesel_demo$ cat diesel.toml
# For documentation on how to configure this file,
# see https://diesel.rs/guides/configuring-diesel-cli

[print_schema]
file = "src/schema.rs"
custom_type_derives = ["diesel::query_builder::QueryId", "Clone"]

[migrations_directory]
dir = "/Users/alizj/work/code/learn-by-doing/rust/diesel_demo/migrations"


zj@a:~/work/code/learn-by-doing/rust/diesel_demo$ ls -l migrations/00000000000000_diesel_initial_setup/
total 8.0K
-rw-r--r-- 1 alizj  328  8  1 14:05 down.sql
-rw-r--r-- 1 alizj 1.2K  8  1 14:05 up.sql


zj@a:~/work/code/learn-by-doing/rust/diesel_demo$ psql -l
                          List of databases
    Name     | Owner | Encoding | Collate | Ctype | Access privileges
-------------+-------+----------+---------+-------+-------------------
 diesel_demo | zj    | UTF8     | C       | C     |
 postgres    | alizj | UTF8     | C       | C     |
 template0   | alizj | UTF8     | C       | C     | =c/alizj         +
             |       |          |         |       | alizj=CTc/alizj
 template1   | alizj | UTF8     | C       | C     | =c/alizj         +
             |       |          |         |       | alizj=CTc/alizj
(4 rows)

zj@a:~/work/code/learn-by-doing/rust/diesel_demo$ cat migrations/00000000000000_diesel_initial_setup/up.sql
-- This file was automatically created by Diesel to setup helper functions
-- and other internal bookkeeping. This file is safe to edit, any future
-- changes will be added to existing projects as new migrations.




-- Sets up a trigger for the given table to automatically set a column called
-- `updated_at` whenever the row is modified (unless `updated_at` was included
-- in the modified columns)
--
-- # Example
--
-- ```sql
-- CREATE TABLE users (id SERIAL PRIMARY KEY, updated_at TIMESTAMP NOT NULL DEFAULT NOW());
--
-- SELECT diesel_manage_updated_at('users');
-- ```
CREATE OR REPLACE FUNCTION diesel_manage_updated_at(_tbl regclass) RETURNS VOID AS $$
BEGIN
    EXECUTE format('CREATE TRIGGER set_updated_at BEFORE UPDATE ON %s
                    FOR EACH ROW EXECUTE PROCEDURE diesel_set_updated_at()', _tbl);
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION diesel_set_updated_at() RETURNS trigger AS $$
BEGIN
    IF (
        NEW IS DISTINCT FROM OLD AND
        NEW.updated_at IS NOT DISTINCT FROM OLD.updated_at
    ) THEN
        NEW.updated_at := current_timestamp;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
zj@a:~/work/code/learn-by-doing/rust/diesel_demo$

zj@a:~/work/code/learn-by-doing/rust/diesel_demo$ cat migrations/00000000000000_diesel_initial_setup/down.sql
-- This file was automatically created by Diesel to setup helper functions
-- and other internal bookkeeping. This file is safe to edit, any future
-- changes will be added to existing projects as new migrations.

DROP FUNCTION IF EXISTS diesel_manage_updated_at(_tbl regclass);
DROP FUNCTION IF EXISTS diesel_set_updated_at();


zj@a:~/work/code/learn-by-doing/rust/diesel_demo$ diesel migration generate create_posts
Creating migrations/2024-08-01-060838_create_posts/up.sql
Creating migrations/2024-08-01-060838_create_posts/down.sql

zj@a:~/work/code/learn-by-doing/rust/diesel_demo$ cat migrations/2024-08-01-060838_create_posts/up.sql
-- Your SQL goes here
CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  title VARCHAR NOT NULL,
  body TEXT NOT NULL,
  published BOOLEAN NOT NULL DEFAULT FALSE
)
zj@a:~/work/code/learn-by-doing/rust/diesel_demo$ cat migrations/2024-08-01-060838_create_posts/down.sql
-- This file should undo anything in `up.sql`
DROP TABLE posts

zj@a:~/work/code/learn-by-doing/rust/diesel_demo$ diesel migration run
Running migration 2024-08-01-060838_create_posts

zj@a:~/work/code/learn-by-doing/rust/diesel_demo$ cat src/schema.rs
// @generated automatically by Diesel CLI.

diesel::table! {
    posts (id) {
        id -> Int4,
        title -> Varchar,
        body -> Text,
        published -> Bool,
    }
}

zj@a:~/work/code/learn-by-doing/rust/diesel_demo$ diesel print-schema
// @generated automatically by Diesel CLI.

diesel::table! {
    posts (id) {
        id -> Int4,
        title -> Varchar,
        body -> Text,
        published -> Bool,
    }
}
```

Sql 语句入口： Module diesel::dsl module，该 module 提供了如下函数：

avg
: Represents a SQL AVG function. This function can only take types which are Foldable.

case_when
: Creates a SQL CASE WHEN ... END expression

`copy_from`
: Creates a COPY FROM statement

`copy_to`
: Creates a COPY TO statement

count
: Creates a SQL COUNT expression

count_distinct
: Creates a SQL COUNT(DISTINCT ...) expression

count_star
: Creates a SQL COUNT(\*) expression

date
: Represents the SQL DATE function. The argument should be a Timestamp expression, and the
    return value will be an expression of type Date.

`delete`
: Creates a DELETE statement.

exists
: Creates a SQL EXISTS expression.

`insert_into`
: Creates an INSERT statement for the target table.

`insert_or_ignore_into`
: Creates an INSERT [OR] IGNORE statement.

max
: Represents a SQL MAX function. This function can only take types which are ordered.

min
: Represents a SQL MIN function. This function can only take types which are ordered.

not
: Creates a SQL NOT expression

`replace_into`
: Creates a REPLACE statement.

`select`
: Creates a bare select statement, with no from clause. Primarily used for testing
    diesel itself, but likely useful for third party crates as well. The given expressions must be
    selectable from anywhere.

sql
: Use literal SQL in the query builder.

`sql_query`
: Construct a full SQL query using raw SQL.

sum
: Represents a SQL SUM function. This function can only take types which are Foldable.

`update`
: Creates an UPDATE statement.


## <span class="section-num">2</span> sql type {#sql-type}

diesel 将 Rust 类型转换为 SQL 类型, 支持的 SQL 类型位于 Module diesel::sql_types。 diesel 为大量
Rust 类型实现了 Trait diesel::deserialize::FromSql trait, 从而在 sql type 和 Rust 类型之间转换, 可以查看 ToSql 和 FromSql trait 的实现文档来确认这个转换过程:

-   支持 Uuid, Json 等类型

<!--listend-->

```rust
// Struct diesel::sql_types::Integer
pub struct Integer;

impl ToSql<Integer, Mysql> for i32
impl ToSql<Integer, Pg> for i32
impl ToSql<Integer, Sqlite> for i32

impl FromSql<Integer, Mysql> for i32
impl FromSql<Integer, Pg> for i32
impl FromSql<Integer, Sqlite> for i32
```


## <span class="section-num">3</span> disel::table! {#disel-table}

```rust
diesel::table! {
    posts (id) {
        id -> Int4,
        title -> Varchar,
        body -> Text,
        published -> Bool,
    }
}
```

生成的代码：

1.  创建了一个 posts module：
    1.  导出了所有与字段同名的类型，如 id、title、body、published；
    2.  嵌套定义一个 dsl module, 也导出了字段同名的类型: posts::dsl::id, posts::dsl::title 等, 以及 posts::dsl::posts 为 posts::table 的别名;
    3.  定义了一个固定的 struct table(空 struct), 但是实现了很多方法:
        -   impl diesel::QuerySource for table
        -   impl &lt;DB&gt;diesel::query_builder::QueryFragment&lt;DB&gt;for table
        -   impl diesel::query_builder::AsQuery for table
        -   impl diesel::Table for table
        -   impl &lt;T&gt;diesel::insertable::Insertable&lt;T&gt;for table

<!--listend-->

```rust
#[allow(unused_imports,dead_code,unreachable_pub)]
pub mod posts {
    pub use self::columns:: * ; // 导出各 field 字段对应的类型,如 id, title, body 等
    use diesel::sql_types:: * ;

    pub mod dsl {
        pub use super::columns::id;
        pub use super::columns::title;
        pub use super::columns::body;
        pub use super::columns::published;
        pub use super::table as posts; // struct table 别名为 posts
    }

    pub const all_columns:(id,title,body,published,) = (id,title,body,published,);

    // 固定类型 table
    pub struct table;

    impl table {
        pub fn star(&self) -> star {
            star // star 是在 self::columns 中定义的 pub struct star;
        }
    }

    pub type SqlType = (Int4,Varchar,Text,Bool,);

    pub type BoxedQuery<'a,DB,ST = SqlType>  = diesel::internal::table_macro::BoxedSelectStatement<'a,ST,diesel::internal::table_macro::FromClause<table> ,DB> ;

    // table 实现了 QuerySource
    impl diesel::QuerySource for table {
        type FromClause = diesel::internal::table_macro::StaticQueryFragmentInstance<table> ;
        type DefaultSelection =  <Self as diesel::Table> ::AllColumns;
        fn from_clause(&self) -> Self::FromClause {
            diesel::internal::table_macro::StaticQueryFragmentInstance::new()
        }
        fn default_selection(&self) -> Self::DefaultSelection {
            use diesel::Table;
            Self::all_columns()
        }
    }

    pub mod columns {
        use super::table;
        use diesel::sql_types:: * ;

        pub struct star;

        // star 实现了各种 trait
        impl <__GB>diesel::expression::ValidGrouping<__GB>for star where(id,title,body,published,):diesel::expression::ValidGrouping<__GB> ,{
            type IsAggregate =  <(id,title,body,published,)as diesel::expression::ValidGrouping<__GB>> ::IsAggregate;
        }

        impl diesel::Expression for star {
            type SqlType = diesel::expression::expression_types::NotSelectable;
        }

        impl <DB:diesel::backend::Backend>diesel::query_builder::QueryFragment<DB>for star where<table as diesel::QuerySource> ::FromClause:diesel::query_builder::QueryFragment<DB> ,{
            #[allow(non_snake_case)]
            fn walk_ast<'b>(&'b self,mut __diesel_internal_out:diesel::query_builder::AstPass<'_,'b,DB>) -> diesel::result::QueryResult<()>{
                use diesel::QuerySource;
                if!__diesel_internal_out.should_skip_from(){
                    const FROM_CLAUSE:diesel::internal::table_macro::StaticQueryFragmentInstance<table>  = diesel::internal::table_macro::StaticQueryFragmentInstance::new();
                    FROM_CLAUSE.walk_ast(__diesel_internal_out.reborrow())? ;
                    __diesel_internal_out.push_sql(".");
                }__diesel_internal_out.push_sql("*");
                Ok(())
            }

        }

        impl diesel::SelectableExpression<table>for star{}

        impl diesel::AppearsOnTable<table>for star{}

        // 每一个 struct 成员对对应一个类型
        #[allow(non_camel_case_types,dead_code)]
        #[derive(Debug,Clone,Copy,diesel::query_builder::QueryId,Default)]
        pub struct id;

        // select  等的 filter() 等使用
        impl diesel::expression::Expression for id {
            type SqlType = Int4;
        }

        impl <DB>diesel::query_builder::QueryFragment<DB>for id

        impl diesel::SelectableExpression<super::table>for id{}

        // delete/update 等的 filter() 等使用
        impl <QS>diesel::AppearsOnTable<QS>for id

        impl <Left,Right>diesel::SelectableExpression<diesel::internal::table_macro::Join<Left,Right,diesel::internal::table_macro::LeftOuter> , >for id

        impl <Left,Right>diesel::SelectableExpression<diesel::internal::table_macro::Join<Left,Right,diesel::internal::table_macro::Inner> , >for id

        impl diesel::query_source::Column for id {
            type Table = super::table;
            const NAME: &'static str = "id";
        }

        // 根据 id 的类型, 自动为它实现了各种 trait
        impl <T>diesel::EqAll<T>for id where T:diesel::expression::AsExpression<Int4> ,diesel::dsl::Eq<id,T::Expression> :diesel::Expression<SqlType = diesel::sql_types::Bool> ,{
            type Output = diesel::dsl::Eq<Self,T::Expression> ;
            fn eq_all(self,__diesel_internal_rhs:T) -> Self::Output {
                use diesel::expression_methods::ExpressionMethods;
                self.eq(__diesel_internal_rhs)
            }

        }
        impl <Rhs> ::std::ops::Add<Rhs>for id where Rhs:diesel::expression::AsExpression< <<id as diesel::Expression> ::SqlType as diesel::sql_types::ops::Add> ::Rhs, > ,{
            type Output = diesel::internal::table_macro::ops::Add<Self,Rhs::Expression> ;
            fn add(self,__diesel_internal_rhs:Rhs) -> Self::Output {
                diesel::internal::table_macro::ops::Add::new(self,__diesel_internal_rhs.as_expression())
            }

        }
```

```rust
fn filter<Predicate>(self, predicate: Predicate) -> Filter<Self, Predicate>
where Self: FilterDsl<Predicate>,

let seans_id = users.filter(name.eq("Sean")).select(id)
    .first(connection);
assert_eq!(Ok(1), seans_id);
let tess_id = users.filter(name.eq("Tess")).select(id)
    .first(connection);
assert_eq!(Ok(2), tess_id);

// 对于 SelectStatement 而言, Predicate 是 Expression
impl<F, S, D, W, O, LOf, G, H, LC, Predicate> FilterDsl<Predicate> for SelectStatement<F, S, D, W, O, LOf, G, H, LC>
where
    Predicate: Expression + NonAggregate,
    Predicate::SqlType: BoolOrNullableBool,
    W: WhereAnd<Predicate>

// 对于 DeleteStatement/UpdateStatement 而言, Predicate 是 AppearsOnTable
impl<T, U, Ret, Predicate> FilterDsl<Predicate> for DeleteStatement<T, U, Ret>
where
    U: WhereAnd<Predicate>,
    Predicate: AppearsOnTable<T>,
    T: QuerySource,

impl<T, U, V, Ret, Predicate> FilterDsl<Predicate> for UpdateStatement<T, U, V, Ret>
where
    T: QuerySource,
    U: WhereAnd<Predicate>,
    Predicate: AppearsOnTable<T>
```

ExpressionMethods 是为所有 Expression 实现的 trait, 故可以用在所有 table field 类型上(因为table!()
宏为它们实现了 Expression)

```rust
// Trait diesel::expression_methods::ExpressionMethods
pub trait ExpressionMethods: Expression + Sized {
    // Provided methods
    fn eq<T>(self, other: T) -> Eq<Self, T>
       where Self::SqlType: SqlType,
             T: AsExpression<Self::SqlType> { ... }
    fn ne<T>(self, other: T) -> NotEq<Self, T>
       where Self::SqlType: SqlType,
             T: AsExpression<Self::SqlType> { ... }
    fn eq_any<T>(self, values: T) -> EqAny<Self, T>
       where Self::SqlType: SqlType,
             T: AsInExpression<Self::SqlType> { ... }
    fn ne_all<T>(self, values: T) -> NeAny<Self, T>
       where Self::SqlType: SqlType,
             T: AsInExpression<Self::SqlType> { ... }
    fn is_null(self) -> IsNull<Self> { ... }
    fn is_not_null(self) -> IsNotNull<Self> { ... }
    fn gt<T>(self, other: T) -> Gt<Self, T>
       where Self::SqlType: SqlType,
             T: AsExpression<Self::SqlType> { ... }
    fn ge<T>(self, other: T) -> GtEq<Self, T>
       where Self::SqlType: SqlType,
             T: AsExpression<Self::SqlType> { ... }
    fn lt<T>(self, other: T) -> Lt<Self, T>
       where Self::SqlType: SqlType,
             T: AsExpression<Self::SqlType> { ... }
    fn le<T>(self, other: T) -> LtEq<Self, T>
       where Self::SqlType: SqlType,
             T: AsExpression<Self::SqlType> { ... }
    fn between<T, U>(self, lower: T, upper: U) -> Between<Self, T, U>
       where Self::SqlType: SqlType,
             T: AsExpression<Self::SqlType>,
             U: AsExpression<Self::SqlType> { ... }
    fn not_between<T, U>(self, lower: T, upper: U) -> NotBetween<Self, T, U>
       where Self::SqlType: SqlType,
             T: AsExpression<Self::SqlType>,
             U: AsExpression<Self::SqlType> { ... }
    fn desc(self) -> Desc<Self> { ... }
    fn asc(self) -> Asc<Self> { ... }
}

// 对于 Expression 实现了 ExpressionMethods
impl<T> ExpressionMethods for T where T: Expression, T::SqlType: SingleValue

// 以 eq() 返回的 Eq 为例, 它实际是  Grouped 类型, 而 Grouped 实现了 Expression trait
pub type Eq<Lhs, Rhs> = Grouped<Eq<Lhs, AsExpr<Rhs, Lhs>>>;
// https://docs.diesel.rs/2.2.x/src/diesel/expression/grouped.rs.html
#[derive(Debug, Copy, Clone, QueryId, Default, DieselNumericOps, ValidGrouping)]
pub struct Grouped<T>(pub T);

impl<T: Expression> Expression for Grouped<T> {
    type SqlType = T::SqlType;
}

impl<T, DB> QueryFragment<DB> for Grouped<T>
where
    T: QueryFragment<DB>,
    DB: Backend + DieselReserveSpecialization,
{
    fn walk_ast<'b>(&'b self, mut out: AstPass<'_, 'b, DB>) -> QueryResult<()> {
        out.push_sql("(");
        self.0.walk_ast(out.reborrow())?;
        out.push_sql(")");
        Ok(())
    }
}
```

综上:

1.  diesel::table!() 宏为 table 的个 field 字段实现了 Expression/AppearsOnTable trait;
2.  所以, 这些 field 字段类型实现了 ExpressionMethods, 可以使用 eq() 方方法;
3.  各 ExpressionMethods 方法返回的类型 Grouped 也实现了 Expression, 所以可以继续链式调用;
4.  各种 xxDsl, 如 FilterDsl 的输入是 Predicate 或 AppearsOnTable, 所以可以传入 ExpressionMethods
    的各种方法返回的对象;

<!--listend-->

```rust
// 示例
let data = users.select(id).filter(name.eq("Sean"));
assert_eq!(Ok(1), data.first(connection));
```


## <span class="section-num">4</span> query {#query}

“Query builder methods” : table!() 宏为 table 实现了 QueryDsl, 故可以在 `table 对象` 上使用 QueryDsl
提供的各种方法来查询数据。

Trait diesel::query_dsl::QueryDsl:

```rust
// Trait diesel::query_dsl::QueryDsl
pub trait QueryDsl: Sized {
    // Provided methods
    fn distinct(self) -> Distinct<Self>
       where Self: DistinctDsl { ... }
    fn distinct_on<Expr>(self, expr: Expr) -> DistinctOn<Self, Expr>
       where Self: DistinctOnDsl<Expr> { ... }
    fn select<Selection>(self, selection: Selection) -> Select<Self, Selection>
       where Selection: Expression, Self: SelectDsl<Selection> { ... }
    fn count(self) -> Select<Self, CountStar>
       where Self: SelectDsl<CountStar> { ... }
    fn inner_join<Rhs>(self, rhs: Rhs) -> InnerJoin<Self, Rhs>
       where Self: JoinWithImplicitOnClause<Rhs, Inner> { ... }
    fn left_outer_join<Rhs>(self, rhs: Rhs) -> LeftJoin<Self, Rhs>
       where Self: JoinWithImplicitOnClause<Rhs, LeftOuter> { ... }
    fn left_join<Rhs>(self, rhs: Rhs) -> LeftJoin<Self, Rhs>
       where Self: JoinWithImplicitOnClause<Rhs, LeftOuter> { ... }
    fn filter<Predicate>(self, predicate: Predicate) -> Filter<Self, Predicate>
       where Self: FilterDsl<Predicate> { ... }
    fn or_filter<Predicate>( self, predicate: Predicate, ) -> OrFilter<Self, Predicate>
       where Self: OrFilterDsl<Predicate> { ... }
    fn find<PK>(self, id: PK) -> Find<Self, PK>
       where Self: FindDsl<PK> { ... }
    fn order<Expr>(self, expr: Expr) -> Order<Self, Expr>
       where Expr: Expression, Self: OrderDsl<Expr> { ... }
    fn order_by<Expr>(self, expr: Expr) -> OrderBy<Self, Expr>
       where Expr: Expression, Self: OrderDsl<Expr> { ... }
    fn then_order_by<Order>(self, order: Order) -> ThenOrderBy<Self, Order>
       where Self: ThenOrderDsl<Order> { ... }
    fn limit(self, limit: i64) -> Limit<Self>
       where Self: LimitDsl { ... }
    fn offset(self, offset: i64) -> Offset<Self>
       where Self: OffsetDsl { ... }
    fn group_by<GB>(self, group_by: GB) -> GroupBy<Self, GB>
       where GB: Expression, Self: GroupByDsl<GB> { ... }
    fn having<Predicate>(self, predicate: Predicate) -> Having<Self, Predicate>
       where Self: HavingDsl<Predicate> { ... }
    fn for_update(self) -> ForUpdate<Self>
       where Self: LockingDsl<ForUpdate> { ... }
    fn for_no_key_update(self) -> ForNoKeyUpdate<Self>
       where Self: LockingDsl<ForNoKeyUpdate> { ... }
    fn for_share(self) -> ForShare<Self>
       where Self: LockingDsl<ForShare> { ... }
    fn for_key_share(self) -> ForKeyShare<Self>
       where Self: LockingDsl<ForKeyShare> { ... }
    fn skip_locked(self) -> SkipLocked<Self>
       where Self: ModifyLockDsl<SkipLocked> { ... }
    fn no_wait(self) -> NoWait<Self>
       where Self: ModifyLockDsl<NoWait> { ... }
    fn into_boxed<'a, DB>(self) -> IntoBoxed<'a, Self, DB>
       where DB: Backend, Self: BoxedDsl<'a, DB> { ... }
    fn single_value(self) -> SingleValue<Self>
       where Self: SingleValueDsl { ... }
    fn nullable(self) -> NullableSelect<Self>
       where Self: SelectNullableDsl { ... }
}

// Table 和 SelectStatement 都实现了 QueryDsl
impl<'a, ST, QS, DB, GB> QueryDsl for BoxedSelectStatement<'a, ST, QS, DB, GB>
impl<F, S, D, W, O, LOf, G, H, LC> QueryDsl for SelectStatement<F, S, D, W, O, LOf, G, H, LC>
impl<S: AliasSource> QueryDsl for Alias<S>
impl<T: Table> QueryDsl for T  // Table 实现了 QueryDsl
```

table 实现的这些方法的返回类型，如 select() 返回的 Select 其实是 SelectDsl 的 Output 类型, 最终这些方法返回的值类型 `都是` `SelectStatement` 类型:

```rust
// 分析 seelct(): 最终返回的是 SelectStatement
pub trait QueryDsl: Sized {
  // ...
  fn select<Selection>(self, selection: Selection) -> Select<Self, Selection>
    where Selection: Expression, Self: SelectDsl<Selection> { ... }
    // ..
}

pub type Select<Source, Selection> = <Source as SelectDsl<Selection>>::Output;

pub trait SelectDsl<Selection: Expression> {
    type Output;

    // Required method
    fn select(self, selection: Selection) -> Self::Output;
}

// https://docs.diesel.rs/2.2.x/src/diesel/query_dsl/select_dsl.rs.html#22-33
impl<T, Selection> SelectDsl<Selection> for T
where
    Selection: Expression,
    T: Table,
    T::Query: SelectDsl<Selection>,
{
    type Output = <T::Query as SelectDsl<Selection>>::Output;

    fn select(self, selection: Selection) -> Self::Output {
        self.as_query().select(selection)
    }
}

// table!() 宏展开后，为 table 实现的 AsQuery trait
// as_query() 返回的 Self::Query 类型是 SelectStatement
impl diesel::query_builder::AsQuery for table {
        type SqlType = SqlType;
        type Query = diesel::internal::table_macro::SelectStatement<diesel::internal::table_macro::FromClause<Self>> ;
        fn as_query(self) -> Self::Query {
            diesel::internal::table_macro::SelectStatement::simple(self)
        }
 }



// 分析 distinct(), 最终返回的还是 SelectStatement
pub trait QueryDsl: Sized {
    fn distinct(self) -> Distinct<Self> where Self: DistinctDsl
    // ...
}

pub type Distinct<Source> = <Source as DistinctDsl>::Output;

pub trait DistinctDsl {
    /// The type returned by `.distinct`
    type Output;

    /// See the trait documentation.
    fn distinct(self) -> dsl::Distinct<Self>;
}

// https://docs.diesel.rs/2.2.x/src/diesel/query_dsl/distinct_dsl.rs.html#26
impl<T> DistinctDsl for T
where
    T: Table + AsQuery<Query = SelectStatement<FromClause<T>>>,
    T::DefaultSelection: Expression<SqlType = T::SqlType> + ValidGrouping<()>,
    T::SqlType: TypedExpressionType,
{
    type Output = dsl::Distinct<SelectStatement<FromClause<T>>>;

    fn distinct(self) -> dsl::Distinct<SelectStatement<FromClause<T>>> {
        self.as_query().distinct()
    }
}
```

Struct diesel::query_builder::SelectStatement 类型代表一个 select 查询：

```rust
#[non_exhaustive]
pub struct SelectStatement<From, Select = DefaultSelectClause<From>, Distinct = NoDistinctClause, Where = NoWhereClause, Order = NoOrderClause, LimitOffset = LimitOffsetClause<NoLimitClause, NoOffsetClause>, GroupBy = NoGroupByClause, Having = NoHavingClause, Locking = NoLockingClause> {
    pub select: Select,
    pub from: From,
    pub distinct: Distinct,
    pub where_clause: Where,
    pub order: Order,
    pub limit_offset: LimitOffset,
    pub group_by: GroupBy,
    pub having: Having,
    pub locking: Locking,
}
```

SelectStatement 实现了各种 diesel::query_dsl::methods module 提供的各种 xxDSL trait， 每个
diesel::query_dsl::methods::xxDSL trait 都提供了一种方法，如 DistinctDsl 提供了 distinct 方法：

BoxedDsl
: The into_boxed method

DistinctDsl
: The distinct method

DistinctOnDsl
: The distinct_on method

ExecuteDsl
: The execute method

FilterDsl
: The filter method

FindDsl
: The find method

GroupByDsl
: The group_by method

HavingDsl
: The having method

LimitDsl
: The limit method

LoadQuery
: The load method

LockingDsl
: Methods related to locking select statements

ModifyLockDsl
: Methods related to modifiers on locking select statements

OffsetDsl
: The offset method

OrFilterDsl
: The or_filter method

OrderDsl
: The order method

SelectDsl
: The select method

SelectNullableDsl
: The nullable method

SingleValueDsl
: The single_value method

ThenOrderDsl
: The then_order_by method

<!--listend-->

```rust
// SelectStatement 实现的 trait

// 实现 QueryDsl，它内部封装了上面其它 xxDsl
impl<F, S, D, W, O, LOf, G, H, LC> QueryDsl for SelectStatement<F, S, D, W, O, LOf, G, H, LC>

// 实现 QueryFragment
impl<F, S, D, W, O, LOf, G, H, LC, DB> QueryFragment<DB> for SelectStatement<F, S, D, W, O, LOf, G, H, LC>
where DB: Backend, Self: QueryFragment<DB, DB::SelectStatementSyntax>

// 实现 QuerySource
impl<From> QuerySource for SelectStatement<From>
where From: AsQuerySource, <From::QuerySource as QuerySource>::DefaultSelection: SelectableExpression<Self>,
```

SelectStatement 也实现了 RunQueryDsl trait，可以用于执行实际的 SQL 查询操作。

“Expression methods”: 各种 QueryDsl 方法的参数主要是 `Expression`, 而 `table!()` 宏为各种 table field
类型实现了 Expression, 而且 Module diesel::expression_methods 为 Expression 提供了各种 method, 这些方法返回的类型也实现了 Expression, 所以可以链式调用:

```rust
// Trait diesel::expression_methods::BoolExpressionMethods
pub trait BoolExpressionMethods: Expression + Sized {
    // Provided methods
    fn and<T, ST>(self, other: T) -> And<Self, T, ST>
       where Self::SqlType: SqlType,
             ST: SqlType + TypedExpressionType,
             T: AsExpression<ST>,
             And<Self, T::Expression>: Expression { ... }
    fn or<T, ST>(self, other: T) -> Or<Self, T, ST>
       where Self::SqlType: SqlType,
             ST: SqlType + TypedExpressionType,
             T: AsExpression<ST>,
             Or<Self, T::Expression>: Expression { ... }
}

// Trait diesel::expression_methods::ExpressionMethods
pub trait ExpressionMethods: Expression + Sized {
    // Provided methods
    fn eq<T>(self, other: T) -> Eq<Self, T>
       where Self::SqlType: SqlType, T: AsExpression<Self::SqlType>
    fn ne<T>(self, other: T) -> NotEq<Self, T>
       where Self::SqlType: SqlType, T: AsExpression<Self::SqlType>
    fn eq_any<T>(self, values: T) -> EqAny<Self, T>
       where Self::SqlType: SqlType, T: AsInExpression<Self::SqlType>
    fn ne_all<T>(self, values: T) -> NeAny<Self, T>
       where Self::SqlType: SqlType, T: AsInExpression<Self::SqlType>
    fn is_null(self) -> IsNull<Self>
    fn is_not_null(self) -> IsNotNull<Self>
    fn gt<T>(self, other: T) -> Gt<Self, T>
       where Self::SqlType: SqlType, T: AsExpression<Self::SqlType>
    fn ge<T>(self, other: T) -> GtEq<Self, T>
       where Self::SqlType: SqlType, T: AsExpression<Self::SqlType>
    fn lt<T>(self, other: T) -> Lt<Self, T>
       where Self::SqlType: SqlType, T: AsExpression<Self::SqlType>
    fn le<T>(self, other: T) -> LtEq<Self, T>
       where Self::SqlType: SqlType, T: AsExpression<Self::SqlType>
    fn between<T, U>(self, lower: T, upper: U) -> Between<Self, T, U>
       where Self::SqlType: SqlType, T: AsExpression<Self::SqlType>, U: AsExpression<Self::SqlType>
    fn not_between<T, U>(self, lower: T, upper: U) -> NotBetween<Self, T, U>
       where Self::SqlType: SqlType, T: AsExpression<Self::SqlType>, U: AsExpression<Self::SqlType>
    fn desc(self) -> Desc<Self>
    fn asc(self) -> Asc<Self>
}

// 示例：
// name 是 Expression，name.eq("Sean") 结果 Eq 还是 Expression
let seans_id = users.filter(name.eq("Sean")).select(id).first(connection);
// species.eq("ferret") 是 Expression，.and() 结果还是 Expression
let data = animals.select((species, name))
    .filter(species.eq("ferret").and(name.eq("Jack")))
    .load(connection)?;
let expected = vec![ (String::from("ferret"), Some(String::from("Jack"))), ];
assert_eq!(expected, data);
```

`Trait diesel::prelude::RunQueryDsl` 根据传入的 Connection 执行实际的 SQL 操作，
SelectStatement/SqlQuery/Alias/SqlLiteral/Table/DeleteStatement/InsertStatement/UpdateStatement 均实现了 RunQueryDsl;

```rust
pub trait RunQueryDsl<Conn>: Sized {
    // Provided methods
    fn execute(self, conn: &mut Conn) -> QueryResult<usize>
       where Conn: Connection, Self: ExecuteDsl<Conn> { ... }
    fn load<'query, U>(self, conn: &mut Conn) -> QueryResult<Vec<U>>
       where Self: LoadQuery<'query, Conn, U> { ... }
    fn load_iter<'conn, 'query: 'conn, U, B>(
        self, conn: &'conn mut Conn, ) -> QueryResult<Self::RowIter<'conn>>
       where U: 'conn, Self: LoadQuery<'query, Conn, U, B> + 'conn { ... }
    fn get_result<'query, U>(self, conn: &mut Conn) -> QueryResult<U>
       where Self: LoadQuery<'query, Conn, U> { ... }
    fn get_results<'query, U>(self, conn: &mut Conn) -> QueryResult<Vec<U>>
       where Self: LoadQuery<'query, Conn, U> { ... }
    fn first<'query, U>(self, conn: &mut Conn) -> QueryResult<U>
       where Self: LimitDsl, Limit<Self>: LoadQuery<'query, Conn, U> { ... }
}

// SelectStatement/SqlQuery/Alias/SqlLiteral/Table/DeleteStatement/InsertStatement/UpdateStatement
// 均实现了 RunQueryDsl：
impl<'a, ST, QS, DB, Conn, GB> RunQueryDsl<Conn> for BoxedSelectStatement<'a, ST, QS, DB, GB>
impl<Conn: Connection, Query> RunQueryDsl<Conn> for BoxedSqlQuery<'_, Conn::Backend, Query>

impl<Inner, Conn> RunQueryDsl<Conn> for SqlQuery<Inner>
impl<Query, Value, Conn> RunQueryDsl<Conn> for UncheckedBind<Query, Value>
impl<S: AliasSource, Conn> RunQueryDsl<Conn> for Alias<S>
impl<ST, T, Conn> RunQueryDsl<Conn> for SqlLiteral<ST, T>
impl<T, Conn> RunQueryDsl<Conn> for T where T: Table // Table

impl<F, S, D, W, O, LOf, G, H, LC, Conn> RunQueryDsl<Conn> for SelectStatement<F, S, D, W, O, LOf, G, H, LC>
impl<T, U, Ret, Conn> RunQueryDsl<Conn> for DeleteStatement<T, U, Ret> where T: QuerySource
impl<T: QuerySource, U, Op, Ret, Conn> RunQueryDsl<Conn> for InsertStatement<T, U, Op, Ret>
impl<T: QuerySource, U, V, Ret, Conn> RunQueryDsl<Conn> for UpdateStatement<T, U, V, Ret>
```

示例：

1.  users table 实现了 QueryDsl，users.select(name) 是 QueryDsl 方法;
2.  传入的 name 实现了 Expression, 返回的 Select 最终类型是 SelectStatement；
3.  SelectStatement 实现了 RunQueryDsl，故可以调用 load() 方法；

<!--listend-->

```rust
diesel::insert_into(users) .values(&vec![name.eq("Sean"); 3]) .execute(connection)?;

// users 是 table 类型对象
let names = users.select(name).load::<String>(connection)?;
let distinct_names = users.select(name).distinct().load::<String>(connection)?;
assert_eq!(vec!["Sean"; 3], names);
assert_eq!(vec!["Sean"; 1], distinct_names);

let count = users.count().get_result(connection);
assert_eq!(Ok(2), count);

let seans_id = users.filter(name.eq("Sean")).select(id).first(connection);
assert_eq!(Ok(1), seans_id);
let tess_id = users.filter(name.eq("Tess")).select(id).first(connection);
assert_eq!(Ok(2), tess_id);

let sean = (1, "Sean".to_string());
let tess = (2, "Tess".to_string());
assert_eq!(Ok(sean), users.find(1).first(connection));
assert_eq!(Ok(tess), users.find(2).first(connection));
assert_eq!(Err::<(i32, String), _>(NotFound), users.find(3).first(connection));

// By default, all columns will be selected
let all_users = users.load::<(i32, String)>(connection)?;
assert_eq!(vec![(1, String::from("Sean")), (2, String::from("Tess"))], all_users);

let all_names = users.select(name).load::<String>(connection)?;
assert_eq!(vec!["Sean", "Tess"], all_names);

// Using a limit
let limited = users.select(name) .order(id) .limit(1) .load::<String>(connection)?;
```

RunQueryDsl 的各方法返回的结果可以保存到实现了 `Selectable 或 Queryable trait` 的自定义类型:

-   Queryable 的 struct 成员字段顺序必须和 table 一致，而 Selectable 可以不一致。
-   \#[diesel(table_name = organization)] 和 #[diesel(check_for_backend(diesel::pg::Pg))] 可以实现编译时静态检查；
-   diesel 为 tuple 也实现了 Queryable，故可以作为方法的返回值类型；

<!--listend-->

```rust
#[derive(Queryable, PartialEq, Debug)]
struct User {
    id: i32,
    name: String,
}
let first_user: User = users.order_by(id).first(connection)?;

#[derive(Identifiable, Queryable, Selectable, Clone, Eq, Hash, PartialEq, Serialize, Deserialize, Debug)]
// 编译时检查，确保 struct 成员和数据库中 table 定义顺序一致
#[diesel(table_name = organization)]
#[diesel(check_for_backend(diesel::pg::Pg))]
pub struct Organization {
    pub id: Uuid,
    pub name: String,
    pub created_at: i64,
    pub updated_at: Option<i64>,
    pub products: Vec<Product>,
}


use schema::{users, posts};
#[derive(Debug, PartialEq, Queryable, Selectable)]
struct User {
    id: i32,
    name: String,
}
#[derive(Debug, PartialEq, Queryable, Selectable)]
#[diesel(table_name = posts)]
struct PostTitle {
    title: String,
}
let (first_user, first_post_title) = users::table
    .inner_join(posts::table)
    .select(<(User, PostTitle)>::as_select())
    .first(connection)?;
let expected_user = User { id: 1, name: "Sean".into() };
assert_eq!(expected_user, first_user);
let expected_post_title = PostTitle { title: "My first post".into() };
assert_eq!(expected_post_title, first_post_title);
```


## <span class="section-num">5</span> Bare functions {#bare-functions}

“Bare functions” : 对于非 select query 类型, 不是直接在 table 对象上调用, diesel 在 Module
diesel::dsl 中提供了相应函数:

avg
: Represents a SQL AVG function. This function can only take types which are Foldable.

case_when
: Creates a SQL CASE WHEN ... END expression

`copy_from`
: Creates a COPY FROM statement

`copy_to`
: Creates a COPY TO statement

count
: Creates a SQL COUNT expression

count_distinct
: Creates a SQL COUNT(DISTINCT ...) expression

count_star
: Creates a SQL COUNT(\*) expression

date
: Represents the SQL DATE function. The argument should be a Timestamp expression, and the
    return value will be an expression of type Date.

`delete`
: Creates a DELETE statement.

exists
: Creates a SQL EXISTS expression.

`insert_into`
: Creates an INSERT statement for the target table.

`insert_or_ignore_into`
: Creates an INSERT [OR] IGNORE statement.

max
: Represents a SQL MAX function. This function can only take types which are ordered.

min
: Represents a SQL MIN function. This function can only take types which are ordered.

not
: Creates a SQL NOT expression

`replace_into`
: Creates a REPLACE statement.

`select`
: Creates a bare select statement, with no from clause. Primarily used for testing
    diesel itself, but likely useful for third party crates as well. The given expressions must be
    selectable from anywhere.

sql
: Use literal SQL in the query builder.

`sql_query`
: Construct a full SQL query using raw SQL.

sum
: Represents a SQL SUM function. This function can only take types which are Foldable.

`update`
: Creates an UPDATE statement.

高亮的这些函数在 diesel crate 中直接导出,故可以使用 diesel::update()

```rust
// update: 注意: 不是在 table 类型对象上调用该函数.
let updated_row = diesel::update(users.filter(id.eq(1)))
    .set((name.eq("James"), surname.eq("Bond")))
    .get_result(connection);
assert_eq!(Ok((1, "James".to_string(), "Bond".to_string())), updated_row);

// delete
let old_count = users.count().first::<i64>(connection);
diesel::delete(users.filter(id.eq(1))).execute(connection)?;
assert_eq!(old_count.map(|count| count - 1), users.count().first(connection));

// insert_into
let rows_inserted = diesel::insert_into(users)
    .values(&name.eq("Sean"))
    .execute(connection);
assert_eq!(Ok(1), rows_inserted);

// select （很少用，一般调用 table 的 select() 方法）
use diesel::dsl::exists;
use diesel::dsl::select;
// exists() 返回 Exists 类型实现了 Expression, 故可以作为 select() 的参数
let sean_exists = select(exists(users.filter(name.eq("Sean")))).get_result(connection);
let jim_exists = select(exists(users.filter(name.eq("Jim")))).get_result(connection);
assert_eq!(Ok(true), sean_exists);
assert_eq!(Ok(false), jim_exists);
```

和 table 实现的 QueryDsl 的 select() 返回的 Select 实际是 `SelectStatement` 类似,
diesel::dsl::select() 或 diesel::select() 函数也返回的是 `SelectStatement` 类型：

```rust
// select() 创建一个  select statement
pub fn select<T>(expression: T) -> select<T> where T: Expression, select<T>: AsQuery

// 返回的 select<T> 其实是 SelectStatement 的类型别名
pub type select<Selection> = SelectStatement<NoFromClause, SelectClause<Selection>>;
```


## <span class="section-num">6</span> delete {#delete}

diesel::delete() 返回 DeleteStatement, 传入的参数类型是 IntoUpdateTarget:

-   Identifiable/table/SelectStatement 均实现了 IntoUpdateTarget，所以 tables 和 tables 调用的各种
    QueryDsl 方法可以为传给 delete()
-   当传入 Identifiable/table 时，删除整个 table 记录。可以传入 SelectStatement 或调用
    DeleteStatement 的 filter方法来限制删除的记录范围。

<!--listend-->

```rust
// Function diesel::dsl::delete
pub fn delete<T: IntoUpdateTarget>( source: T, ) -> DeleteStatement<T::Table, T::WhereClause>

// A type which can be passed to update or delete.
pub trait IntoUpdateTarget: HasTable {
    type WhereClause;
    // Required method
    fn into_update_target(self) -> UpdateTarget<Self::Table, Self::WhereClause>;
}

// Struct diesel::query_builder::UpdateTarget
pub struct UpdateTarget<Table, WhereClause> {
    pub table: Table,
    pub where_clause: WhereClause,
}

// Identifiable 和 table 实现了 IntoUpdateTarget
impl<T, Tab, V> IntoUpdateTarget for T
where
    T: Identifiable<Table = Tab>,
    Tab: Table + FindDsl<T::Id>,
    Find<Tab, T::Id>: IntoUpdateTarget<Table = Tab, WhereClause = V>

impl IntoUpdateTarget for table

// SelectStatement 也实现了 IntoUpdateTarget （但是奇怪的是，生成的 docs 并没有显示 ）
// https://docs.diesel.rs/2.2.x/src/diesel/query_builder/select_statement/dsl_impls.rs.html#540
impl<F, W> IntoUpdateTarget
    for SelectStatement<FromClause<F>, DefaultSelectClause<FromClause<F>>, NoDistinctClause, W>
where
    F: QuerySource,
    Self: HasTable,
    W: ValidWhereClause<F>,
{
    type WhereClause = W;

    fn into_update_target(self) -> UpdateTarget<Self::Table, Self::WhereClause> {
        UpdateTarget {
            table: Self::table(),
            where_clause: self.where_clause,
        }
    }
}

// 示例
let deleted_rows = diesel::delete(users) // 传入实现 IntoUpdateTarget 的 table
    .filter(name.eq("Sean"))
    .execute(connection);
assert_eq!(Ok(1), deleted_rows);

let old_count = users.count().first::<i64>(connection);
diesel::delete(users.filter(id.eq(1))) // 传入实现 IntoUpdateTarget 的 SelectStatement
    .execute(connection)?;
assert_eq!(old_count.map(|count| count - 1), users.count().first(connection));
```

Struct diesel::query_builder::DeleteStatement：

-   DeleteStatement 实现了 filter/or_filter/returning 方法;
-   returning() 的参数 SelectableExpression， table field 和它的各种 tuple 类型实现了该 trait。

<!--listend-->

```rust
pub struct DeleteStatement<T: QuerySource, U, Ret = NoReturningClause> { /* private fields */ }

// DeleteStatement 实现了 filter/or_filter/returning 方法
impl<T: QuerySource, U> DeleteStatement<T, U, NoReturningClause>

pub fn filter<Predicate>(self, predicate: Predicate) -> Filter<Self, Predicate>
  where Self: FilterDsl<Predicate>

// Calling foo.filter(bar).or_filter(baz) is identical to foo.filter(bar.or(baz)).
pub fn or_filter<Predicate>( self, predicate: Predicate, ) -> OrFilter<Self, Predicate>
  where Self: OrFilterDsl<Predicate>,

// table!() 宏生成的 filed 字段类型都实现了 SelectableExpression，可以用作 returning 的参数。
//
// 如果要返回多个字段，则需要使用 tuple 类型。
pub fn returning<E>( self, returns: E, ) -> DeleteStatement<T, U, ReturningClause<E>>
  where E: SelectableExpression<T>, DeleteStatement<T, U, ReturningClause<E>>: Query,

// 示例
let deleted_rows = diesel::delete(users)
    .filter(name.eq("Sean"))
    .execute(connection);
assert_eq!(Ok(1), deleted_rows);
let expected_names = vec!["Tess".to_string()];
let names = users.select(name).load(connection);
assert_eq!(Ok(expected_names), names);

let deleted_rows = diesel::delete(users)
    .filter(name.eq("Sean"))
    .or_filter(name.eq("Tess"))
    .execute(connection);
assert_eq!(Ok(2), deleted_rows);
let num_users = users.count().first(connection);
assert_eq!(Ok(0), num_users);

let deleted_name = diesel::delete(users.filter(name.eq("Sean")))
    // users table 的 name 字段类型实现了 SelectableExpression，如果要返回多个字段，则需要使用 tuple 类型。
    // 然后在 get_result::<(Type1, Type2)>(connection) 中指定返回值类型。
    .returning(name)
    .get_result(connection);
assert_eq!(Ok("Sean".to_string()), deleted_name);
```

关于 retuning 的示例：

```rust
let inserted_user = insert_into(users)
        .values(new_users)
        .returning((name, hair_color))
        .get_result::<(String, Option<String>)>(connection)
        .unwrap();
    let expected_user = ("Sean".to_string(), Some("Black".to_string()));
```


## <span class="section-num">7</span> insert {#insert}

diesel::insert_into() 或 diesel::dsl::insert_into() 函数：

-   values() 方法的参数类型是 Insertable：table/&amp;SelectStatement/Vec/tuple 均实现了它，同时
    \#[derive(Insertable)] 修饰的 struct 类型也实现了它。

<!--listend-->

```rust
// target 参数类型是 table
pub fn insert_into<T: Table>(target: T) -> IncompleteInsertStatement<T>

// IncompleteInsertStatement 实现了 default_values 和 values 方法：
pub struct IncompleteInsertStatement<T, Op = Insert> { /* private fields */ }
impl<T, Op> IncompleteInsertStatement<T, Op> where T: QuerySource
pub fn default_values(self) -> InsertStatement<T, DefaultValues, Op>
pub fn values<U>(self, records: U) -> InsertStatement<T, U::Values, Op> where U: Insertable<T>,

// table 和 &SelectStatement 均实现了 Insertable trait，所以 table!() 生成表字段及其 ExpressMehtod
// 结果都可以作为 values() 方法的参数。
impl<'a, F, S, D, W, O, LOf, G, H, LC, Tab> Insertable<Tab> for &'a SelectStatement<F, S, D, W, O, LOf, G, H, LC>
where
    Tab: Table,
    Self: Query,
    <Tab::AllColumns as ValidGrouping<()>>::IsAggregate: MixedAggregates<No, Output = No>,

impl<T> Insertable<T> for table
where
    <table as AsQuery>::Query: Insertable<T>,

// Vec/Option/[T;N] 均实现了 Insertable
impl<T, Tab> Insertable<Tab> for Vec<T>
where
    T: Insertable<Tab> + UndecoratedInsertRecord<Tab>,
type Values = BatchInsert<Vec<<T as Insertable<Tab>>::Values>, Tab, (), false>
fn values(self) -> Self::Values

impl<T, Tab, V> Insertable<Tab> for Option<T>
where
    T: Insertable<Tab, Values = ValuesClause<V, Tab>>,
    InsertableOptionHelper<T, V>: Insertable<Tab>,
type Values = <InsertableOptionHelper<T, V> as Insertable<Tab>>::Values
fn values(self) -> Self::Values


impl<T, Tab, const N: usize> Insertable<Tab> for [T; N]
where
    T: Insertable<Tab>,
impl<T, Tab, const N: usize> Insertable<Tab> for Box<[T; N]>
where
    T: Insertable<Tab>,
type Values = BatchInsert<Vec<<T as Insertable<Tab>>::Values>, Tab, [<T as Insertable<Tab>>::Values; N], true>
fn values(self) -> Self::Values
```

-   You may add data by calling `values()` or `default_values()` as shown in the examples.
-   Backends that support the `RETURNING clause`, such as PostgreSQL, can return `the inserted rows` by
    calling `.get_results` instead of `.execute`.

<!--listend-->

```rust
// 例子
diesel::sql_query("CREATE TABLE users (
    name VARCHAR(255) NOT NULL DEFAULT 'Sean',
    hair_color VARCHAR(255) NOT NULL DEFAULT 'Green'
)").execute(connection)?;
insert_into(users) // users 是 Table
    .default_values()
    .execute(connection)
    .unwrap();
let inserted_user = users.first(connection)?;
let expected_data = (String::from("Sean"), String::from("Green"));
assert_eq!(expected_data, inserted_user);

let rows_inserted = diesel::insert_into(users) // users 是 Table
    .values(&name.eq("Sean")) // name.eq() 返回的 &SelectStatement 实现了 Insertable
    .execute(connection);
assert_eq!(Ok(1), rows_inserted);

let new_users = vec![
    name.eq("Tess"),
    name.eq("Jim"),
];
let rows_inserted = diesel::insert_into(users)
    .values(&new_users) // Vec 也实现了 Insertable
    .execute(connection);
assert_eq!(Ok(2), rows_inserted);

// Using a tuple for values
let new_user = (id.eq(1), name.eq("Sean"));
let rows_inserted = diesel::insert_into(users)
    .values(&new_user) // tuple 也实现了 Insertable
    .execute(connection);
assert_eq!(Ok(1), rows_inserted);

let new_users = vec![
    (id.eq(2), name.eq("Tess")),
    (id.eq(3), name.eq("Jim")),
];
let rows_inserted = diesel::insert_into(users)
    .values(&new_users)
    .execute(connection);
assert_eq!(Ok(2), rows_inserted);


// Using struct for values
#[derive(Insertable)] // 自动实现 Insertable，该 struct 可以作为 values() 的参数
#[diesel(table_name = users)]
struct NewUser<'a> {
    name: &'a str,
}

// Insert one record at a time
let new_user = NewUser { name: "Ruby Rhod" };
diesel::insert_into(users)
    .values(&new_user)
    .execute(connection)
    .unwrap();
// Insert many records
let new_users = vec![
    NewUser { name: "Leeloo Multipass" },
    NewUser { name: "Korben Dallas" },
];
let inserted_names = diesel::insert_into(users)
    .values(&new_users)
    .execute(connection)
    .unwrap();

// Inserting default value for a column
// You can use Option<T> to allow a column to be set to the default value when needed.
// When the field is set to None, diesel inserts the default value on supported databases. When the field is set to Some(..), diesel inserts the given value.
// The column color in brands table is NOT NULL DEFAULT 'Green'.
#[derive(Insertable)]
#[diesel(table_name = brands)]
struct NewBrand {
    color: Option<String>, // 缺省值
}
// Insert `Red`
let new_brand = NewBrand { color: Some("Red".into()) };
diesel::insert_into(brands)
    .values(&new_brand)
    .execute(connection)
    .unwrap();
// Insert the default color
let new_brand = NewBrand { color: None };
diesel::insert_into(brands)
    .values(&new_brand)
    .execute(connection)
    .unwrap();

#[derive(Insertable)]
#[diesel(table_name = brands)]
struct NewBrand {
    accent: Option<Option<String>>,
}

// Insert `Red`
let new_brand = NewBrand { accent: Some(Some("Red".into())) };

diesel::insert_into(brands)
    .values(&new_brand)
    .execute(connection)
    .unwrap();

// Insert the default accent
let new_brand = NewBrand { accent: None };

diesel::insert_into(brands)
    .values(&new_brand)
    .execute(connection)
    .unwrap();

// Insert `NULL`
let new_brand = NewBrand { accent: Some(None) };

diesel::insert_into(brands)
    .values(&new_brand)
    .execute(connection)
    .unwrap();

// Insert from select
let new_posts = users::table
    .select((
        users::name.concat("'s First Post"),
        users::id,
    ));
diesel::insert_into(posts::table)
    .values(new_posts)
    .into_columns((posts::title, posts::user_id))
    .execute(conn)?;

let inserted_posts = posts::table
    .select(posts::title)
    .load::<String>(conn)?;
let expected = vec!["Sean's First Post", "Tess's First Post"];
assert_eq!(expected, inserted_posts);

// With return value
let inserted_names = diesel::insert_into(users)
    .values(&vec![
        name.eq("Diva Plavalaguna"),
        name.eq("Father Vito Cornelius"),
    ])
    .returning(name)
    .get_results(connection);
assert_eq!(Ok(vec!["Diva Plavalaguna".to_string(), "Father Vito Cornelius".to_string()]), inserted_names);
```

Trait diesel::prelude::Insertable

-   可以使用 derived macro 来自动实现：

<!--listend-->

```rust
pub trait Insertable<T> {
    type Values;

    // Required method
    fn values(self) -> Self::Values;

    // Provided method
    fn insert_into(self, table: T) -> InsertStatement<T, Self::Values>
       where T: Table,
             Self: Sized { ... }
}
```

Struct diesel::query_builder::InsertStatement：

-   提供的方法：into_columns()/returning()/on_conflict_do_nothing()/on_conflict();
-   实现了 ExecuteDsl 和 RunQueryDsl，执行实际的 SQL 操作。

<!--listend-->

```rust
#[non_exhaustive]
pub struct InsertStatement<T: QuerySource, U, Op = Insert, Ret = NoReturningClause> {
    pub operator: Op,
    pub target: T,
    pub records: U,
    pub returning: Ret,
    /* private fields */
}

impl<T: QuerySource, U, Op, Ret> InsertStatement<T, U, Op, Ret>
pub fn new(target: T, records: U, operator: Op, returning: Ret) -> Self
pub fn into_columns<C2>( self, columns: C2, ) -> InsertStatement<T, InsertFromSelect<U, C2>, Op, Ret>
   where C2: ColumnList<Table = T> + Expression, U: Query<SqlType = C2::SqlType>
pub fn returning<E>( self, returns: E,) -> InsertStatement<T, U, Op, ReturningClause<E>>
   where InsertStatement<T, U, Op, ReturningClause<E>>: Query
pub fn on_conflict_do_nothing( self, ) -> InsertStatement<T, OnConflictValues<U::ValueClause, NoConflictTarget, DoNothing<T>>, Op, Ret>
pub fn on_conflict<Target>( self, target: Target, ) -> IncompleteOnConflict<InsertStatement<T, U::ValueClause, Op, Ret>, ConflictTarget<Target>> where ConflictTarget<Target>: OnConflictTarget<T>,


impl<V, T, QId, C, Op, O, const STATIC_QUERY_ID: bool> ExecuteDsl<C, Sqlite> for InsertStatement<T, BatchInsert<Vec<ValuesClause<V, T>>, T, QId, STATIC_QUERY_ID>, Op>
where
    T: QuerySource,
    C: Connection<Backend = Sqlite>,
    V: ContainsDefaultableValue<Out = O>,
    O: Default,
    (O, Self): ExecuteDsl<C, Sqlite>,
fn execute(query: Self, conn: &mut C) -> QueryResult<usize>
impl<T: QuerySource, U, Op, Ret, Conn> RunQueryDsl<Conn> for InsertStatement<T, U, Op, Ret>
impl<T: Copy + QuerySource, U: Copy, Op: Copy, Ret: Copy> Copy for InsertStatement<T, U, Op, Ret>
where T::FromClause: Copy,
```

on_conflict() 方法返回的 IncompleteOnConflict 类型提供了 do_nothing() 和 do_update() 方法：

```rust
pub struct IncompleteOnConflict<Stmt, Target> { /* private fields */ }

pub fn do_nothing( self, ) -> InsertStatement<T, OnConflictValues<U, Target, DoNothing<T>>, Op, Ret>
pub fn do_update(self) -> IncompleteDoUpdate<Stmt, Target>

pub struct IncompleteDoUpdate<Stmt, Target> { /* private fields */ }
// IncompleteDoUpdate 提供了 set() 方法；
pub fn set<Changes>( self, changes: Changes,
) -> InsertStatement<T, OnConflictValues<U, Target, DoUpdate<Changes::Changeset, T>>, Op, Ret>
where
    T: QuerySource, Changes: AsChangeset<Target = T>,

// 示例
let user = User { id: 1, name: "Sean" };

let user_count = users.count().get_result::<i64>(conn)?;
assert_eq!(user_count, 0);

diesel::insert_into(users)
    .values(&user)
    .on_conflict_do_nothing()
    .execute(conn)?;
let user_count = users.count().get_result::<i64>(conn)?;
assert_eq!(user_count, 1);

let user = User { id: 1, name: "Pascal" };
let user2 = User { id: 1, name: "Sean" };
assert_eq!(Ok(1), diesel::insert_into(users).values(&user).execute(conn));
let insert_count = diesel::insert_into(users)
    .values(&user2)
    .on_conflict(id)
    .do_update()
    .set(name.eq("I DONT KNOW ANYMORE"))
    .execute(conn);
assert_eq!(Ok(1), insert_count);
assert_eq!(Ok(2), insert_count);
let users_in_db = users.load(conn);
assert_eq!(Ok(vec![(1, "I DONT KNOW ANYMORE".to_string())]), users_in_db);
```


## <span class="section-num">8</span> update {#update}

diesel::update() 或 diesel::dsl::update():

-   传入的是 IntoUpdateTarget，Identifiable/table/SelectStatement 实现了该 trait（参考 delete 部分）

<!--listend-->

```rust
pub fn update<T: IntoUpdateTarget>( source: T, ) -> UpdateStatement<T::Table, T::WhereClause>
```

Struct diesel::query_builder::UpdateStatement

-   方法：set/filter/returning；

<!--listend-->

```rust
pub struct UpdateStatement<T: QuerySource, U, V = SetNotCalled, Ret = NoReturningClause> { /* private fields */ }

// 实现的方法
impl<T: QuerySource, U> UpdateStatement<T, U, SetNotCalled>
pub fn set<V>(self, values: V) -> UpdateStatement<T, U, V::Changeset>
  where
    T: Table,
    V: AsChangeset<Target = T>, // 传入的 values 是 AsChangeset
    UpdateStatement<T, U, V::Changeset>: AsQuery

pub fn filter<Predicate>(self, predicate: Predicate) -> Filter<Self, Predicate>
  where Self: FilterDsl<Predicate>,impl<T: QuerySource, U, V, Ret> UpdateStatement<T, U, V, Ret>

pub fn returning<E>(self, returns: E,) -> UpdateStatement<T, U, V, ReturningClause<E>>
  where
    T: Table,
    UpdateStatement<T, U, V, ReturningClause<E>>: Query,

// 示例
let updated_rows = diesel::update(users)
    .set(name.eq("Jim")) // Eq 实现了 AsChangeset
    .filter(name.eq("Sean"))
    .execute(connection);
assert_eq!(Ok(1), updated_rows);
let expected_names = vec!["Jim".to_string(), "Tess".to_string()];
let names = users.select(name).order(id).load(connection);
assert_eq!(Ok(expected_names), names);

let updated_name = diesel::update(users.filter(id.eq(1))) // 传入的是 SelectStatement
    .set(name.eq("Dean"))
    .returning(name)
    .get_result(connection);
assert_eq!(Ok("Dean".to_string()), updated_name);
```

set() 的参数是 Trait diesel::query_builder::AsChangeset

-   可以被 derived 生成；
-   Eq 和 Grouped 实现了 AsChangeset；
-   如果要设置多个值，可以使用 tuple。

<!--listend-->

```rust
pub trait AsChangeset {
    type Target: QuerySource;
    type Changeset;

    // Required method
    fn as_changeset(self) -> Self::Changeset;
}

impl<T: AsChangeset> AsChangeset for Option<T>

// https://docs.diesel.rs/2.2.x/src/diesel/query_builder/update_statement/changeset.rs.html#41
//
// Eq 和 Grouped 也实现了 AsChangeset
impl<Left, Right> AsChangeset for Eq<Left, Right> where Left: AssignmentTarget, Right: AppearsOnTable<Left::Table>

impl<Left, Right> AsChangeset for Grouped<Eq<Left, Right>> where Eq<Left, Right>: AsChangeset,
```


## <span class="section-num">9</span> sql_query {#sql-query}

diesel::sql_query() 或 diesel::dsl::sql_query()

```rust
pub fn sql_query<T: Into<String>>(query: T) -> SqlQuery
```

使用 raw SQL 来构造完整的查询，如果部分使用 raw SQL 则可以用 sql().

```rust
let users = sql_query("SELECT * FROM users ORDER BY id").load(connection);
let expected_users = vec![
    User { id: 1, name: "Sean".into() },
    User { id: 2, name: "Tess".into() },
];
assert_eq!(Ok(expected_users), users);

// Checkout the documentation of your database for the correct bind placeholder
let users = sql_query("SELECT * FROM users WHERE id > ? AND name <> ?");
let users = users
    .bind::<Integer, _>(1)
    .bind::<Text, _>("Tess")
    .get_results(connection);
let expected_users = vec![
    User { id: 3, name: "Jim".into() },
];
assert_eq!(Ok(expected_users), users);
```

raw SQL 中的参数需要使用 SqlQuery::bind() 来设置。

Struct diesel::query_builder::SqlQuery

-   方法: bind()/sql()

<!--listend-->

```rust
pub struct SqlQuery<Inner = Empty> { /* private fields */ }

impl<Inner> SqlQuery<Inner>
pub fn bind<ST, Value>(self, value: Value) -> UncheckedBind<Self, Value, ST>
// Appends a piece of SQL code at the end.
pub fn sql<T: AsRef<str>>(self, sql: T) -> Self

// 示例
let users = sql_query("SELECT * FROM users WHERE id > ? AND name <> ?")
    .bind::<Integer, _>(1)
    .bind::<Text, _>("Tess")
    .get_results(connection);
let expected_users = vec![
    User { id: 3, name: "Jim".into() },
];
assert_eq!(Ok(expected_users), users);
```
