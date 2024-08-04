---
title: "diesel"
author: ["zhangjun"]
lastmod: 2024-08-04T17:09:18+08:00
tags: ["rust"]
categories: ["rust"]
draft: false
series: ["rust crate"]
series_order: 18
---

## <span class="section-num">1</span> 安装 postgresql 和 diesel_cli {#安装-postgresql-和-diesel-cli}

```shell
$ brew install mysql-client sqlite postgresql
$ ls -l /opt/homebrew/opt/mysql-client/lib
total 14M
-rw-r--r-- 1 alizj 6.6M  8  1 13:44 libmysqlclient.23.dylib
-r--r--r-- 1 alizj 7.1M 12 14  2023 libmysqlclient.a
lrwxr-xr-x 1 alizj   23 12 14  2023 libmysqlclient.dylib -> libmysqlclient.23.dylib
drwxr-xr-x 3 alizj   96  8  1 13:44 pkgconfig/

$ export MYSQLCLIENT_LIB_DIR=/opt/homebrew/opt/mysql-client/lib MYSQLCLIENT_VERSION=23
$ cargo install diesel_cli

# 开启启动 postgresql
$ brew services start postgresql@14

# 确认启动成功
$ brew services list
Name          Status  User  File
bind          none
dbus          none
emacs-plus@30 none
postgresql@14 started alizj ~/Library/LaunchAgents/homebrew.mxcl.postgresql@14.plist
unbound       none

$ lsof -i tcp:5432
COMMAND    PID  USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
postgres 64220 alizj    7u  IPv6 0x5297ba1d8b4e0e13      0t0  TCP localhost:postgresql (LISTEN)
postgres 64220 alizj    8u  IPv4 0x5297ba271f55c663      0t0  TCP localhost:postgresql (LISTEN)

# 连接 postgresql，设置账号密码
$ psql -d postgres
psql (14.12 (Homebrew))
Type "help" for help.
postgres=# CREATE ROLE zj WITH LOGIN PASSWORD '1234';
CREATE ROLE
postgres=# ALTER ROLE zj CREATEDB;
ALTER ROLE

$ psql -l
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

Rust 和 Diesel 类型列表：
<https://kotiri.com/2018/01/31/postgresql-diesel-rust-types.html>


## <span class="section-num">3</span> derive macro {#derive-macro}

列表：<https://docs.diesel.rs/2.2.x/diesel/prelude/index.html#derives>

AsChangeset
: Implements AsChangeset
    -   diesel::insert(table).on_conflict().set() 的参数类型是 AsChangeset, Eq/Grouped&lt;Eq&lt;Left,
        Right&gt;&gt;/Tuple 均实现了该 trait, 同时对于自定义类型 struct 实现该 trait 后, 可以传入自定义类型.


Associations
: Implement required traits for the associations API
    -   用于在关联表场景，在 child 表（含有外键的表）上定义。同时使用的还有
        \#[diesel(belongs_to(Book, foreign_key = book_id))] 来指定关联表和外键字段。


Identifiable
: Implements Identifiable for references of the current type
    -   Identifiable 实现了 IntoUpdateTarget，所以可以将自定义 struct 作为 diesel::delete()/update()
        的参数；


Insertable
: Implements Insertable
    -   为 struct 类型实现 Insertable，这样可以用于 diesel::insert_into(table).values() 的参数


Queryable
: Implements Queryable to load the result of statically typed queries
    -   从 SQL query 中创建一个 struct 类型对象;
    -   可以作为 diesel::prelude::RunQueryDsl 的各方法返回值类型, 例如:
        users::table.inner_join(posts::table).select(&lt;(User,
        PostTitle)&gt;::as_select()).first(connection)?;


QueryableByName
: Implements QueryableByName for untyped sql queries, such as that one
    generated by sql_query


Selectable
: Implements Selectable
    -   根据 struct model 和 #[diesel(table_name = XX)] 指定的 table 构造一个 select clause；
    -   和 Queryable 类似, 可以作为 diesel::prelude::RunQueryDsl 的各方法返回值类型; 但是可以只指定
        table 的 `部分字段而非全部` ;
    -   Selectable 实现了 SelectableHelper trait, 后者提供了:
        1.  as_returning() 方法, 可以用作 DeleteStatement/SelectStatement/UpdateStatement 的
            retuning() 方法的参数, 从而返回一个 struct.
        2.  as_select() 方法, 可以用作 SelectStatement 的 select() 方法的参数, 从而返回一个 struct.

<!--listend-->

```rust
use diesel::prelude::*;

#[derive(Queryable, Selectable)]
// table_name 是 Selectable 所需要的，默认为 struct 全小写名称加 s。
// check_for_backend 用于编译时静态检查，即检查 struct 成员类型和数据库表定义是否完全一致。
#[diesel(table_name = crate::schema::posts)]
#[diesel(check_for_backend(diesel::pg::Pg))]
pub struct Post {
    pub id: i32, // 默认表的 PK 是 id 字段。
    pub title: String,
    pub body: String,
    pub published: bool,
}

// 用于 diesel::insert_into(table).values() 的参数，需要实现 Insertable
#[derive(Insertable)]
#[diesel(table_name = posts)]
pub struct NewPost<'a> {
    pub title: &'a str,
    pub body: &'a str,
}

let new_post = NewPost { title, body };
diesel::insert_into(posts::table)
    .values(&new_post)
    .returning(Post::as_returning()) // Selectable 可以作为 returning() 的参数
    .get_result(conn)
    .expect("Error saving new post");
let post = posts
    .find(post_id)
    .select(Post::as_select()) // Selectable 可以作为 select() 的参数
    .first(connection)
    .optional(); // This allows for returning an Option<Post>, otherwise it will throw an error

use schema::{users, posts};
#[derive(Debug, PartialEq, Queryable, Selectable)]
struct User {
    id: i32,
    name: String,
}
#[derive(Debug, PartialEq, Queryable, Selectable)]
struct Post {
    id: i32,
    user_id: i32,
    title: String,
}
let (first_user, first_post) = users::table
    .inner_join(posts::table)
    .select(<(User, Post)>::as_select()) // tuple 也实现了 Selectable
    .first(connection)?;


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
#[derive(Debug, PartialEq, Queryable, Selectable)]
struct UserPost {
    #[diesel(embed)]
    user: User, // 支持嵌入字段
    #[diesel(embed)]
    post_title: PostTitle,
}
let first_user_post = users::table
    .inner_join(posts::table)
    .select(UserPost::as_select())
    .first(connection)?;

let expected_user_post = UserPost {
    user: User {
        id: 1,
        name: "Sean".into(),
    },
    post_title: PostTitle {
        title: "My first post".into(),
    },
};
assert_eq!(expected_user_post, first_user_post);

// 示例: Identifiable 可以作为 update() 的参数, AsChangeset 可以作为 set() 的参数.  Identifiable
// 需要有作为 PK 的 id 字段.
#[derive(Queryable, Identifiable, AsChangeset)]
#[diesel(table_name = posts)]
pub struct Post {
    pub id: i64,
    pub title: String,
    pub body: String,
    pub draft: bool,
    pub publish_at: SystemTime,
    pub visit_count: i32,
}
diesel::update(post) // post 是上面的实现了 Identifiable 的 Post 类型值, 必须要设置 id 字段.
    .set(posts::draft.eq(false))
    .execute(conn)

update(post)
// 等效于
update(posts.find(post.id))
// 或
update(posts.filter(id.eq(post.id))).

// Associations 关联表
#[derive(Queryable, Identifiable, Selectable, Debug, PartialEq)]
#[diesel(table_name = books)]
pub struct Book {
    pub id: i32,
    pub title: String,
}
#[derive(Queryable, Selectable, Identifiable, Associations, Debug, PartialEq)]
#[diesel(belongs_to(Book, foreign_key = book_id))] // foreign_key 默认为 {table}_id 字段
#[diesel(table_name = pages)]
pub struct Page {
    pub id: i32,
    pub page_number: i32,
    pub content: String,
    pub book_id: i32,
}
```


## <span class="section-num">4</span> disel::table! {#disel-table}

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

1.  生成 3 个嵌套的 module: posts, posts::dsl 和 posts::columns;
2.  posts::columns module 定义了:
    -   pub struct star; 它实现了各种 trait;
    -   每个数据表成员对应一个同名 struct 类型, 如 pub struct id; 它也实现了各种 trait;
3.  posts::dsl module:
    -   导入了 posts::columns 中定义的 id 等各种表字段的同名类型, 将 table 重命名为 posts;

<!--listend-->

```rust
#[allow(unused_imports,dead_code,unreachable_pub)]
pub mod posts {
    pub use self::columns:: * ; // 导出各 field 字段对应的类型,如 id, title, body 等

    pub mod dsl {
        pub use super::columns::id;
        pub use super::columns::title;
        pub use super::columns::body;
        pub use super::columns::published;
        pub use super::table as posts; // table 别名为 posts
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

    pub type BoxedQuery<'a,DB,ST = SqlType>

    // table 实现了各种 trait
    impl diesel::QuerySource for table
    impl <DB>diesel::query_builder::QueryFragment<DB>for table
    impl diesel::query_builder::AsQuery for table
    impl diesel::Table for table
    impl diesel::associations::HasTable for table
    impl diesel::query_builder::IntoUpdateTarget for table
    impl diesel::query_source::AppearsInFromClause<table>for table
    impl <T>diesel::insertable::Insertable<T>for table

    #[doc = r" Contains all of the columns of this table"]
    pub mod columns {
        use super::table;

        // 代表 users.* 查询
        pub struct star;
        // star 实现了各种 trait
        impl <__GB>diesel::expression::ValidGrouping<__GB>for star
        impl diesel::Expression for star {
        impl <DB:diesel::backend::Backend>diesel::query_builder::QueryFragment<DB>for star
        impl diesel::SelectableExpression<table>for star
        impl diesel::AppearsOnTable<table>for star

        // 每一个 struct 成员对对应一个类型
        #[allow(non_camel_case_types,dead_code)]
        #[derive(Debug,Clone,Copy,diesel::query_builder::QueryId,Default)]
        pub struct id;

        // 每个成员类型都实现了 Expression, 在 select 中使用.
        impl diesel::expression::Expression for id {
            type SqlType = Int4; // 决定了 ExpressionMethods 可以使用的方法
        }

        impl <DB>diesel::query_builder::QueryFragment<DB>for id
        impl diesel::SelectableExpression<super::table>for id

        // delete/update 等的 filter() 等使用
        impl <QS>diesel::AppearsOnTable<QS>for id

        impl <Left,Right>diesel::SelectableExpression<diesel::internal::table_macro::Join<Left,Right,diesel::internal::table_macro::LeftOuter> , >for id

        impl diesel::query_source::Column for id

        // 根据 id 的类型, 自动为它实现了各种 trait
        impl <T>diesel::EqAll<T>for id
        impl <Rhs> ::std::ops::Add<Rhs>for id
        impl <Rhs> ::std::ops::Sub<Rhs>for id
        impl <Rhs> ::std::ops::Div<Rhs>for id
        impl <Rhs> ::std::ops::Mul<Rhs>for id
```

常用方式:

1.  posts::table 或 posts::dsl::posts;
2.  posts::id/title 或 posts::dsl::id/title

一般情况下, 在函数内部导入 posts::dsl::\*,  一般不使用 posts::columns::id 的形式

```rust
// 不建议:
users::table
    .filter(users::name.eq("Sean"))
    .filter(users::hair_color.eq("black"))

    // 建议: 常用的形式:
    users.filter(name.eq("Sean")).filter(hair_color.eq("black"))

// 如果要使用 star, 则可以调用 users 的 star() 方法

// 示例:
fn main() {
    use self::schema::posts::dsl::posts; // 导入 posts table
    // ...
    let post = posts
        .find(post_id)
        .select(Post::as_select())
        .first(connection)
        .optional(); // This allows for returning an Option<Post>, otherwise it will throw an error
    // ...
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
       where Self::SqlType: SqlType,
             T: AsExpression<Self::SqlType>,
             U: AsExpression<Self::SqlType>
    fn not_between<T, U>(self, lower: T, upper: U) -> NotBetween<Self, T, U>
       where Self::SqlType: SqlType,
             T: AsExpression<Self::SqlType>,
             U: AsExpression<Self::SqlType>
    fn desc(self) -> Desc<Self>
    fn asc(self) -> Asc<Self>
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


## <span class="section-num">5</span> query {#query}

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
// 分析 select(): 最终返回的是 SelectStatement
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
diesel::query_dsl::methods::xxDSL trait 都提供了一种方法，如 DistinctDsl 提供了 distinct 方法, 它们的结果还是 SelectStatement 类型, 故可以链式调用：

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
: The select method, 指定要返回的字段 tuple 或实现 Selectable 的 struct

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


#[derive(Queryable, Selectable)]
#[diesel(table_name = crate::schema::posts)]
#[diesel(check_for_backend(diesel::pg::Pg))]
pub struct Post {
    pub id: i32,
    pub title: String,
    pub body: String,
    pub published: bool,
}
let post = posts
    .find(post_id)
    .select(Post::as_select()) // Selectable 提供了 as_select() 方法
    .first(connection)
    .optional(); // This allows for returning an Option<Post>, otherwise it will throw an error
```


## <span class="section-num">6</span> debug_query {#debug-query}

函数 diesel::debug_query 用于将 QueryFragment 查询表达式打印出来（实现了 fmt::Display 和
fmt::Debug）, 常用于显示 diesel 语句调试。

```rust
pub fn debug_query<DB, T>(query: &T) -> DebugQuery<'_, T, DB>

let sql = debug_query::<DB, _>(&users.count()).to_string();
assert_eq!(sql, "SELECT COUNT(*) FROM `users` -- binds: []");

let query = users.find(1);
let debug = debug_query::<DB, _>(&query);
assert_eq!(debug.to_string(), "SELECT `users`.`id`, `users`.`name` FROM `users` \
    WHERE (`users`.`id` = ?) -- binds: [1]");

let debug = format!("{:?}", debug);
let expected = "Query { \
    sql: \"SELECT `users`.`id`, `users`.`name` FROM `users` WHERE \
        (`users`.`id` = ?)\", \
    binds: [1] \
}";
assert_eq!(debug, expected);let sql = debug_query::<DB, _>(&users.count()).to_string();
assert_eq!(sql, "SELECT COUNT(*) FROM `users` -- binds: []");

let query = users.find(1);
let debug = debug_query::<DB, _>(&query);
assert_eq!(debug.to_string(), "SELECT `users`.`id`, `users`.`name` FROM `users` \
    WHERE (`users`.`id` = ?) -- binds: [1]");

let debug = format!("{:?}", debug);
let expected = "Query { \
    sql: \"SELECT `users`.`id`, `users`.`name` FROM `users` WHERE \
        (`users`.`id` = ?)\", \
    binds: [1] \
}";
assert_eq!(debug, expected);
```


## <span class="section-num">7</span> RunQueryDsl {#runquerydsl}

`Trait diesel::prelude::RunQueryDsl` 根据传入的 Connection 执行实际的 SQL 操作，
SelectStatement/SqlQuery/Alias/SqlLiteral/Table/DeleteStatement/InsertStatement/UpdateStatement 均实现了 RunQueryDsl, 用于执行实际的 SQL 操作.

-   execute/load() 返回实际影响的计数条数;
-   get_result/get_results/first() 返回插入/更新后的值, 如果没有调用 returning() 方法, 则返回表的所有字段.
-   get_result() 返回 0 个记录是表示出错, 如果要返回 0 或 1 个记录, 需要使用
    get_result(...).optional();

<!--listend-->

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

RunQueryDsl 的 get_result/get_results/first() 方法返回的结果可以保存到实现了 `Selectable 或
Queryable trait` 的自定义类型, 或者 tuple 中:

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

如果自定义类型实现了 #[derive(AsChangeset)] and #[derive(Identifiable)], 则可以使用该类型的
save_changes() 方法:

```rust
foo.save_changes(&conn)
// 等效于
diesel::update(&foo).set(&foo).get_result(&conn).
```


## <span class="section-num">8</span> bare functions {#bare-functions}

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


## <span class="section-num">9</span> delete {#delete}

diesel::delete() 返回 DeleteStatement, 传入的参数类型是 IntoUpdateTarget:

-   Identifiable/table/SelectStatement 均实现了 IntoUpdateTarget，所以 tables 和 tables 调用的各种
    QueryDsl 方法可以为传给 delete()
-   当传入 Identifiable/table 时，删除整个 table 记录。可以传入 SelectStatement 或调用
    DeleteStatement 的 filter方法来限制删除的记录范围。
-   Identifiable 可以通过 derive macro 为自定义 struct 类型来实现，这样可以 delete() 可以传入自定义类型。

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

// Identifiable 实现了 IntoUpdateTarget，故可以传入实现了 Identifiable 的自定义 struct 类型
impl<T, Tab, V> IntoUpdateTarget for T
where
    T: Identifiable<Table = Tab>,
    Tab: Table + FindDsl<T::Id>,
    Find<Tab, T::Id>: IntoUpdateTarget<Table = Tab, WhereClause = V>

// table 实现了 IntoUpdateTarget
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
-   returning() 的参数 SelectableExpression，table field 和它的各种 tuple 类型实现了该 trait。

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


## <span class="section-num">10</span> insert {#insert}

diesel::insert_into() 或 diesel::dsl::insert_into() 函数：

-   values() 方法的参数类型是 Insertable：table/&amp;SelectStatement/Vec/tuple 均实现了它;
-   \#[derive(Insertable)] 修饰的 struct 类型也实现了它，所以可以传入自定义 struct 类型；

<!--listend-->

```rust
// target 参数类型是 table
pub fn insert_into<T: Table>(target: T) -> IncompleteInsertStatement<T>

pub struct IncompleteInsertStatement<T, Op = Insert> { /* private fields */ }

// IncompleteInsertStatement 实现了 default_values 和 values 方法：
impl<T, Op> IncompleteInsertStatement<T, Op> where T: QuerySource
pub fn default_values(self) -> InsertStatement<T, DefaultValues, Op>
pub fn values<U>(self, records: U) -> InsertStatement<T, U::Values, Op> where U: Insertable<T>,

// table 和 &SelectStatement 均实现了 Insertable trait，所以 table!() 生成表字段及其 ExpressMehtod
// 结果都可以作为 values() 方法的参数。
impl<'a, F, S, D, W, O, LOf, G, H, LC, Tab> Insertable<Tab> for &'a SelectStatement

impl<T> Insertable<T> for table

// Vec/Option/[T;N] 均实现了 Insertable
impl<T, Tab> Insertable<Tab> for Vec<T> where T: Insertable<Tab> + UndecoratedInsertRecord<Tab>

impl<T, Tab, V> Insertable<Tab> for Option<T>
where
    T: Insertable<Tab, Values = ValuesClause<V, Tab>>, InsertableOptionHelper<T, V>: Insertable<Tab>,

impl<T, Tab, const N: usize> Insertable<Tab> for [T; N] where T: Insertable<Tab>

impl<T, Tab, const N: usize> Insertable<Tab> for Box<[T; N]> where T: Insertable<Tab>
```

如果表格所有字段都有缺省值，调用 default_values() 方法：

```rust
use schema::users::dsl::*;
insert_into(users).default_values().execute(conn)
// 等效于: INSERT INTO "users" DEFAULT VALUES -- binds: []
```

如果要实际设置值, 使用 values() 方法:

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
    .values(&new_user) // tuple 也实现了 Insertable, 一次插入多个字段值
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
#[derive(Insertable)] // 自动实现 Insertable，该 struct 可以作为 values() 的参数, 实现一次更新多个字段值
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

对于支持 `RETURNING clause` 的数据库, 如 PostgreSQL 和 SQLite, 可以调用 .get_results() 来获取插入的记录.

```rust
let inserted_users = insert_into(users)
    .values(&vec![
        (id.eq(1), name.eq("Sean")),
        (id.eq(2), name.eq("Tess")),
    ])
    .get_results(conn)?;
// 等效于:
// INSERT INTO "users" ("id", "name") VALUES ($1, $2), ($3, $4)
// RETURNING "users"."id", "users"."name", "users"."hair_color",
//           "users"."created_at", "users"."updated_at"
// -- binds: [1, "Sean", 2, "Tess"]

let expected_users = vec![
    User {
        id: 1,
        name: "Sean".into(),
        hair_color: None,
        created_at: now,
        updated_at: now,
    },
    User {
        id: 2,
        name: "Tess".into(),
        hair_color: None,
        created_at: now,
        updated_at: now,
    },
];
assert_eq!(expected_users, inserted_users);
```

get_result/get_results() 默认然会所有列, 使用 returning() 来指定要返回的列:

```rust
use schema::users::dsl::*;

insert_into(users)
    .values(name.eq("Ruby"))
    .returning(id)
    .get_result(conn)

// 等效于:
// INSERT INTO "users" ("name") VALUES ($1)
// RETURNING "users"."id"
// -- binds: ["Ruby"]
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

// 示例
#[derive(Insertable)]
#[diesel(table_name = posts)]
pub struct NewPost<'a> {
    pub title: &'a str,
    pub body: &'a str,
}
let new_post = NewPost { title, body };
diesel::insert_into(posts::table)
    .values(&new_post)
    .returning(Post::as_returning()) // Selectable 提供了 as_returning
    .get_result(conn)
    .expect("Error saving new post")
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

-   do_update() 返回的对象的 set() 方法的参数是 `AsChangeset` 类型, 可以使用 derive macro 来实现, 这样可以传入自定义 struct 类型;
-   Eq/Grouped&lt;Eq&lt;Left,Right&gt;&gt; 和 Tuple 也实现了 AsChangeset;

<!--listend-->

```rust
pub struct IncompleteOnConflict<Stmt, Target> { /* private fields */ }

pub fn do_nothing( self, ) -> InsertStatement<T, OnConflictValues<U, Target, DoNothing<T>>, Op, Ret>

pub fn do_update(self) -> IncompleteDoUpdate<Stmt, Target>
pub struct IncompleteDoUpdate<Stmt, Target> { /* private fields */ }
// IncompleteDoUpdate 提供了 set() 方法,参数是 AsChangeset 类型
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
    .set(name.eq("I DONT KNOW ANYMORE")) // 参数是 AsChangeset 类型, Eq 实行了该 trait
    .execute(conn);
assert_eq!(Ok(1), insert_count);
assert_eq!(Ok(2), insert_count);
let users_in_db = users.load(conn);
assert_eq!(Ok(vec![(1, "I DONT KNOW ANYMORE".to_string())]), users_in_db);
```


## <span class="section-num">11</span> update {#update}

diesel::update() 或 diesel::dsl::update():

-   传入的是 IntoUpdateTarget 类型, Identifiable/table/SelectStatement 实现了该 trait（参考 delete
    部分）

<!--listend-->

```rust
pub fn update<T: IntoUpdateTarget>( source: T, ) -> UpdateStatement<T::Table, T::WhereClause>
```

Struct diesel::query_builder::UpdateStatement

-   方法：set/filter/returning()；

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
    .filter(name.eq("Sean")) // 限制 update 的范围
    .execute(connection);
assert_eq!(Ok(1), updated_rows);

let updated_name = diesel::update(users.filter(id.eq(1))) // 传入的是 SelectStatement, 限制 update 范围
    .set(name.eq("Dean"))
    .returning(name)
    .get_result(connection);
assert_eq!(Ok("Dean".to_string()), updated_name);

let post = diesel::update(posts.find(id))
    .set(published.eq(true))
    .returning(Post::as_returning())
    .get_result(connection)
    .unwrap();
println!("Published post {}", post.title);
```

set() 的参数是 Trait diesel::query_builder::AsChangeset 类型：

-   可以被 derived 生成，这样可以使用自定义 struct 来作为参数；
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

impl<Left, Right> AsChangeset for Grouped<Eq<Left, Right>> where Eq<Left, Right>: AsChangeset


// 示例: Identifiable 可以作为 update() 的参数, AsChangeset 可以作为 set() 的参数(一次更新多个字段).
// Identifiable 需要有作为 PK 的 id 字段.
#[derive(Queryable, Identifiable, AsChangeset)]
#[diesel(table_name = posts)]
#[diesel(primary_key(id))] // 如果 PK 不是 id, 则需要明确指定
pub struct Post {
    pub id: i64,
    pub title: String,
    pub body: String,
    pub draft: bool,
    pub publish_at: SystemTime,
    pub visit_count: i32,
}
diesel::update(post) // post 是上面的实现了 Identifiable 的 Post 类型值, 必须要设置 id 字段
    .set(posts::draft.eq(false))
    .execute(conn)

update(post)
// 等效于
update(posts.find(post.id))
// 或
update(posts.filter(id.eq(post.id))).

diesel::update(posts)
    .set(visit_count.eq(visit_count + 1)) // 更新单个字段
    .execute(conn)

diesel::update(posts)
    .set(( // 更新多个字段值, 可以使用 tuple
        title.eq("[REDACTED]"),
        body.eq("This post has been classified"),
    ))
    .execute(conn)

// 通过实现 AsChangetset, 更新多个字段时, 可以传入 post struct 值类型
diesel::update(posts::table).set(post).execute(conn)


#[derive(AsChangeset)]
#[diesel(table_name = posts)]
// #[diesel(treat_none_as_null = true)]  // 将 None 设置为 NULL
struct PostForm<'a> {
    title: Option<&'a str>, // 可选字段
    body: Option<&'a str>,
}

diesel::update(posts::table)
    .set(&PostForm {
        title: None, // None 表示不设置该字段
        body: Some("My new post"),
    })
    .execute(conn)
// 等效于: UPDATE "posts" SET "body" = $1 -- binds: ["My new post"]
```


## <span class="section-num">12</span> sql_query {#sql-query}

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


## <span class="section-num">13</span> associations/join {#associations-join}


### <span class="section-num">13.1</span> diesel::associations {#diesel-associations}

module diesel::associations 提供了表之间 1-N 的关联关系宏和函数.

Associations in Diesel are always child-to-parent.

在 child 表上使用 #[derive(Associations)] 和 #[diesel(belongs_to(Parent))] 来定义与 Parent 之间的关联关系:

-   Foreigin Key 惯例: table_name_id, 如果不是这样, 需要明确指定: #[diesel(belongs_to(Foo,
    foreign_key = mykey))].

<!--listend-->

```rust
use schema::{posts, users};

#[derive(Identifiable, Queryable, PartialEq, Debug)]
#[diesel(table_name = users)]
pub struct User {
    id: i32,
    name: String,
}

#[derive(Identifiable, Queryable, Associations, PartialEq, Debug)]
#[diesel(belongs_to(User))]
#[diesel(table_name = posts)]
pub struct Post {
    id: i32,
    user_id: i32, // Foreigin Key 惯例: table_name_id
    title: String,
}

let user = users.find(2).get_result::<User>(connection)?;
let users_post = Post::belonging_to(&user)
    .first(connection)?;
let expected = Post { id: 3, user_id: 2, title: "My first post too".into() };
assert_eq!(expected, users_post);
```

使用 child 的 belonging_to() 方法来查询一个或多个 parent 的 child 记录：

```rust
let sean = users.filter(name.eq("Sean")).first::<User>(connection)?;
let tess = users.filter(name.eq("Tess")).first::<User>(connection)?;

let seans_posts = Post::belonging_to(&sean)
    .select(title)
    .load::<String>(connection)?;
assert_eq!(vec!["My first post", "About Rust"], seans_posts);

// A vec or slice can be passed as well
let more_posts = Post::belonging_to(&vec![sean, tess])
    .select(title)
    .load::<String>(connection)?;
assert_eq!(vec!["My first post", "About Rust", "My first post too"], more_posts);
```

可以对查询结果 Vec&lt;Child&gt; 使用 grouped_by() 来进行聚合，参数是 &amp;[Parent], 返回的是按 Parent 聚合后的Vec&lt;Vec&lt;Child&gt;&gt;：

```rust
let users = users::table.load::<User>(connection)?;
let posts = Post::belonging_to(&users)
    .load::<Post>(connection)?
    .grouped_by(&users);
let data = users.into_iter().zip(posts).collect::<Vec<_>>();

let expected_data = vec![
    (
        User { id: 1, name: "Sean".into() },
        vec![
            Post { id: 1, user_id: 1, title: "My first post".into() },
            Post { id: 2, user_id: 1, title: "About Rust".into() },
        ],
    ),
    (
        User { id: 2, name: "Tess".into() },
        vec![
            Post { id: 3, user_id: 2, title: "My first post too".into() },
        ],
    ),
];

assert_eq!(expected_data, data);
```


### <span class="section-num">13.2</span> 1-N {#1-n}

1-N 即 belong to 关系，module diesel::associations 提供了支撑。

创建两个表的 migration:

```shell
diesel migration generate create_books
diesel migration generate create_pages
```

up 语句:

```sql
CREATE TABLE books (
  id SERIAL PRIMARY KEY,
  title VARCHAR NOT NULL
);

CREATE TABLE pages (
  id SERIAL PRIMARY KEY,
  page_number INT NOT NULL,
  content TEXT NOT NULL,
  book_id INTEGER NOT NULL REFERENCES books(id)
);
```

执行 migration:

```shell
diesel migration run
```

生成的 schema.rs:

-   diesel::joinable!() 宏的输入参数为: child_table -&gt; parent_table (foreign_key), 只能适用于一个
    foreign_key 的情况, 对于其它情况(如 composite foreign key) 在查询时需要通过 ON clause 来指定;

<!--listend-->

```rust
// @generated automatically by Diesel CLI.

diesel::table! {
    books (id) {
        id -> Int4,
        title -> Varchar,
    }
}

diesel::table! {
    pages (id) {
        id -> Int4,
        page_number -> Int4,
        content -> Text,
        book_id -> Int4,
    }
}

diesel::joinable!(pages -> books (book_id));

diesel::allow_tables_to_appear_in_same_query!(
    books,
    pages,
);
```

diesel::joinable!() 宏可以消除在关联查询时使用 ON cluase 的情况:

```rust
use schema::*;

// Child table: posts
// Parent table: users
// Foreign key: user_id (in child table: posts)
joinable!(posts -> users (user_id));
allow_tables_to_appear_in_same_query!(posts, users);

// 消除 ON clause
let implicit_on_clause = users::table.inner_join(posts::table);
let implicit_on_clause_sql = diesel::debug_query::<DB, _>(&implicit_on_clause).to_string();

// 显式使用 ON clause
let explicit_on_clause = users::table
    .inner_join(posts::table.on(posts::user_id.eq(users::id)));
let explicit_on_clause_sql = diesel::debug_query::<DB, _>(&explicit_on_clause).to_string();

// 两者是等价的
assert_eq!(implicit_on_clause_sql, explicit_on_clause_sql);

// posts JOIN users ON posts.user_id = users.id
```

数据模型 src/model.rs:

```rust
use diesel::prelude::*;

use crate::schema::{books, pages};

#[derive(Queryable, Identifiable, Selectable, Debug, PartialEq)]
#[diesel(table_name = books)]
pub struct Book {
    pub id: i32,
    pub title: String,
}

// Child 表添加 Associations
#[derive(Queryable, Selectable, Identifiable, Associations, Debug, PartialEq)]
// belongs_to 指定 Parent table, 如果是约定的 {Parent}_id 作为 FK, 则可以不指定
#[diesel(belongs_to(Book, foreign_key = book_id))]
#[diesel(table_name = pages)]
pub struct Page {
    pub id: i32,
    pub page_number: i32,
    pub content: String,
    pub book_id: i32,
}
```

读数据:

```rust
let momo = books::table
    .filter(books::title.eq("Momo"))
    .select(Book::as_select())
    .get_result(conn)?;

// belonging_to() 是 1-N 关联查询, 传入的可以是单个 Parent 或多个 Parent.
let pages = Page::belonging_to(&momo)
    .select(Page::as_select())
    .load(conn)?; // 使用 load() 而非 execute()/get_results(), 故可以添加额外的 clause
//指定的语句: SELECT * FROM pages WHERE book_id IN(…)
println!("Pages for \"Momo\": \n {pages:?}\n");


// 查询所有的 books
let all_books = books::table.select(Book::as_select()).load(conn)?;
// get all pages for all books
let pages = Page::belonging_to(&all_books) // bool slice
    .select(Page::as_select())
    .load(conn)?; // 不实际查询, 可以添加额外的 clause
// group the pages per book
let pages_per_book = pages
    .grouped_by(&all_books)
    .into_iter()
    .zip(all_books)
    .map(|(pages, book)| (book, pages)) // 返回一个 tuple
    .collect::<Vec<(Book, Vec<Page>)>>();
println!("Pages per book: \n {pages_per_book:?}\n");
```

反序列化结果到自定义类型:

```rust
// [{
//     "id": 1,
//     "title": "Momo",
//     "pages": […],
// }]

#[derive(Serialize)] // serde 提供的 Serialize macro
struct BookWithPages {
    #[serde(flatten)] // 将 book 字段内容打平到结果类型中(默认是位于 book 字段中)
    book: Book,
    pages: Vec<Page>,
}

// group the pages per book
let pages_per_book = pages
    .grouped_by(&all_books)
    .into_iter()
    .zip(all_books)
    .map(|(pages, book)| BookWithPages { book, pages })
    .collect::<Vec<BookWithPages>>();
```


### <span class="section-num">13.3</span> join {#join}

对于非 1-N 的关联关系，diesel 需要使用 SQL JOIN 来解决。diesel 提供了两种类型 join：INNER JOIN 和
LEFT JOIN。

QueryDsl::inner_join() 用于构建 INNER JOIN 语句：

-   如果没有使用 select(), 则默认返回一个 tuple，包含双方的所有默认字段；select() 的参数可以是 tuple
    或实现了 Queryable 的类型；

<!--listend-->

```rust
let page_with_book = pages::table
    .inner_join(books::table)
    .filter(books::title.eq("Momo"))
    .select((Page::as_select(), Book::as_select()))
    .load::<(Page, Book)>(conn)?;
println!("Page-Book pairs: {page_with_book:?}");

// 两种不同的 inner_join() 类型：
users::table.inner_join(posts::table.inner_join(comments::table));
// Results in the following SQL
// SELECT * FROM users
// INNER JOIN posts ON users.id = posts.user_id
// INNER JOIN comments ON post.id = comments.post_id

users::table.inner_join(posts::table).inner_join(comments::table);
// Results in the following SQL
// SELECT * FROM users
// INNER JOIN posts ON users.id = posts.user_id
// INNER JOIN comments ON users.id = comments.user_id
```

left_join():

-   返回的结果类型是：(Book, Option&lt;Page&gt;)

<!--listend-->

```rust
let book_without_pages = books::table
    .left_join(pages::table)
    .select((Book::as_select(), Option::<Page>::as_select()))
    .load::<(Book, Option<Page>)>(conn)?;
println!("Book-Page pairs (including empty books): {book_without_pages:?}");
```

默认使用 joinable!() 来隐式构建 ON 语句，也可以使用 JoinOnDsl::on() 来指定自定义 join 规则：

```rust
pub trait JoinOnDsl: Sized {
    // Provided method
    fn on<On>(self, on: On) -> On<Self, On> { ... }
}

let data = users::table
    .left_join(posts::table.on(
        users::id.eq(posts::user_id).and(
            posts::title.eq("My first post"))
    ))
    .select((users::name, posts::title.nullable()))
    .load(connection);
let expected = vec![
    ("Sean".to_string(), Some("My first post".to_string())),
    ("Tess".to_string(), None),
];
assert_eq!(Ok(expected), data);
```


### <span class="section-num">13.4</span> M-N {#m-n}

Many-to-Many 需要使用 Join Table 来实现，它 belongs_to 所有关联的 table。

创建 migration：

```shell
diesel migration generate create_authors
diesel migration generate create_books_authors // 创建 join table
```

join table 的定义中要使用 FK 来引用关联的表记录：

```sql
CREATE TABLE authors (
  id SERIAL PRIMARY KEY,
  name VARCHAR NOT NULL
);

CREATE TABLE books_authors (
  book_id INTEGER REFERENCES books(id),
  author_id INTEGER REFERENCES authors(id),
  PRIMARY KEY(book_id, author_id)
);
```

执行迁移：diesel migration run

创建 model：

-   核心是 join table 要 belongs_to 到所有的关联的表；

<!--listend-->

```rust
use diesel::prelude::*;

use crate::schema::{books, pages, authors, books_authors};

#[derive(Queryable, Selectable, Identifiable, PartialEq, Debug)]
#[diesel(table_name = authors)]
pub struct Author {
    pub id: i32,
    pub name: String,
}

#[derive(Identifiable, Selectable, Queryable, Associations, Debug)]
#[diesel(belongs_to(Book))]
#[diesel(belongs_to(Author))]
#[diesel(table_name = books_authors)]
#[diesel(primary_key(book_id, author_id))]
pub struct BookAuthor {
    pub book_id: i32,
    pub author_id: i32,
}
```

读数据：

-   先使用 belonging_to() 来进行 1-N join，然后再使用 inner_join();

<!--listend-->

```rust
let astrid_lindgren = authors::table
    .filter(authors::name.eq("Astrid Lindgren"))
    .select(Author::as_select())
    .get_result(conn)?;

// get all of Astrid Lindgren's books
let books = BookAuthor::belonging_to(&astrid_lindgren)
    .inner_join(books::table)
    .select(Book::as_select())
    .load(conn)?;
println!("Books by Astrid Lindgren: {books:?}");

// 更复杂的例子
let all_authors = authors::table
    .select(Author::as_select())
    .load(conn)?;

let books = BookAuthor::belonging_to(&authors)
    .inner_join(books::table)
    .select((BookAuthor::as_select(), Book::as_select()))
    .load(conn)?;

let books_per_author: Vec<(Author, Vec<Book>)> = books
    .grouped_by(&all_authors)
    .into_iter()
    .zip(authors)
    .map(|(b, author)| (author, b.into_iter().map(|(_, book)| book).collect()))
    .collect();

println!("All authors including their books: {books_per_author:?}");
```
