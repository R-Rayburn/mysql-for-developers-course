# mysql-for-developers-course
Course over mysql basics and information for building schemas, creating indexes, and optimizing queries.
Couse site: https://planetscale.com/learn/courses/mysql-for-developers
## Schema
### Intro
- When creating schemas, always pick the correct type, and the smallest type. This allows queries to easily search through data due to less space consumption.
- Reducing the complexity and making sure the data matches what you need also helps with compacting and improving DB interaction.
### INT
There are five data types that can store integers:
| Type       | Storage (Bytes) | Minimum Signed    | Maximum Signed   | Minimum Unsigned | Maximum Unsigned |
|------------|-----------------|-------------------|------------------|------------------|------------------|
| TINYINT    | 1               |              -128 | 127              | 0                | 255              |
| SMALLINT   | 2               |           -32,768 | 32,767           | 0                | 65,535           |
| MEDIUMNINT | 3               |        -8,388,608 | 8,388,607        | 0                | 16,777,215       |
| INT        | 4               |    -2,147,483,648 | 2,147,483,647    | 0                | 42,949,672,295   |
| BIGINT     | 8               |             -2^63 | 2^63 - 1         | 0                | 2^64 - 1         |

Use for **Int()**:
When using Int(11), this doesn't control the range of numbers, but rather the width of bytes used. This is a deprecated functionality that will be removed in the near future and shouldn't be used.

Understanding the different types of integer columns and their ranges is crucial to designing a database schema that's both efficient and scalable.
By selecting the appropriate data type, we can avoid unnecessary storage requirements and ensure efficient queries.

### DECIMAL
Fixed Point (Exact Value)
- DECIMAL (NUMERIC)

Floating Point (Approximate Value)
- FLOAT
- DOUBLE

Example of side-effects of floating:
```sql

CREATE TABLE decimals (
  n INT,
  d1 DOUBLE,
  d2 DOUBLE
);

INSERT INTO
  decimals
VALUES
  (1, 100.40, 20.40),
  (1, -80.00, 0.00);

SELECT * FROM decimals;

-- 1 | 100.4 | 20.4
-- 1 |   -80 |    0

SELECT SUM(d1), SUM(d2) FROM decimals GROUP BY n;
-- Returns an approximate value
-- 20.400000000000006 | 20.4

```


Example of fixed decimal:
```sql

CREATE TABLE decimals (
  n INT,
  -- max of 10 digets with 2 after the decimal (00000000.00)
  d1 DECIMAL(10, 2),
  d2 DECIMAL(10, 2)
);

INSERT INTO
  decimals
VALUES
  (1, 100.40, 20.40),
  (1, -80.00, 0.00);

SELECT * FROM decimals;

-- 1 | 100.40 | 20.40
-- 1 | -80.00 |  0.00

SELECT SUM(d1), SUM(d2) FROM decimals GROUP BY n;
-- Returns an exact value
-- 20.40 | 20.40

```

**FLOAT** is 4 bytes

**DOUBLE** is 8 bytes

When storing values that require absolute precision (currency, financial data, etc) you should use the DECIMAL data type.
DECIMAL allows you to specify the exact number of digits including decimal points.
This can be done by using the syntax ```DECIMAL(10,2)``` where the first number is the number of digits, and the second is the number of decimal places to reserve.
The result of the above example will give us numbers that fill in `00000000.00` with the fixed percision of 2 decimal places, and 8 digits before the decimal.

If using a data type for scientific calculations, where relative precision is more important than absolute precision, consider using FLOAT or DOUBLE.

When working with decimal values in MySQL, it's important to choose the
correct data type for your needs. If you require absolute precision in
your values, DECIMAL is the best choice. If you don't require exact
values, FLOAT or DOUBLE may be more appropriate, depending on the range
and amount of percision you need. With FLOAT or DOUBLE, values might be
slightly different than what is expected due to them not being exactly
percise.

### STRING
All available string types:
Talk about now
- CHAR    (fixed character)
- VARCHAR (variable character)

Discuss later
- TINYTEXT
- TEXT
- MEDIUMTEXT
- LONGTEXT
- BINARY
- VARBINARY
- TINYBLOB
- BLOB
- MEDIUMBLOB
- LONGBLOB
- ENUM
- SET


```sql
CREATE TABLE strings (
  fixed5 CHAR(5),
  fixed32 CHAR(32),
  fixed100 CHAR(100), -- "Robert                   ..." always takes 100 bytes
  var100 VARCHAR(100) -- "Robert" takes 6 bytes + 1 byte to prefix how long it is
)
```

`CHAR` column will awlays take up all space, while `VARCHAR` column will only take up the space it requires, up to the amount allocated.

#### `CHARSET` and `COLATE`:

`CHARSET` tells you what characters (A, a, B, b, ...) are allowed in the column.
All values for `CHARSET` can be found with 
```sql
SELECT * FROM information_schema.CHARACTER_SETS ORDER BY CHARACTER_SET_NAME;
```

`COLATE` are rules that are used to determine how to compare two strings (A == a, a > A, ...).

`ci` means **case incinsitive** (A === a)

`ai` means **accent incinsitive** (e === é)

### Binary Strings

-- CHAR -- BINARY (fixed)

-- VARCHAR -- VARBINARY (variable)

These columns (BINARY and VARBINARY) only store bytes in their columns. No Character Set, no Colation.

```sql
CREATE TABLE bins (
  bin BINARY(16),
  var VARBINARY(100)
);

SELECT unhex(md5('hello')), md5('hello');

INSERT INTO bins VALUES (UNHEX(MD5('hello')), UNHEX(MD5('hello')))

SELECT * FROM bins;
```

> NOTE: When connecting to a mysql server, you can pass in the `--skip-binary-as-hex` flag so that you are able to view raw binary strings that may be stored.

These can be useful when storing things like hashes that you use. This is because you don't need things like character sets and collation for hashes, and the data size is smaller than a `CHAR`. When wanting to select these, you can run a query like
```sql
-- bin_hash BINARY(16)
SELECT * FROM bins WHERE bin_hash = UNHEX('<hash goes here>')
```

`BINARY` and `VARBINARY` data types are similar to `CHAR` and `VARCHAR` without the constraints to following the rules set for character sets and collations. This is because these two binary data types only store bytes that represent the data being stored.

### Long Strings

Character data types (has character set and colation):
- CHAR
- VARCHAR
- TEXT (TINYTEXT (255 characters), TEXT, MEDIUMTEXT, LONGTEXT (4GB))

Binary data types (has no character set or colation):
- BINARY
- VARBINARY
- BLOB (TINYBLOB, BLOB, MEDIUMBLOB, LONGBLOB (4GB))

Do not select tables with blobs unless you want the blobs (same for medium and long texts)
```sql
SELECT * FROM table_with_blobs;
```
Instead, separate the data out and join in when you want to include that larger information, or reference exact columns. you are looking at.
```sql
SELECT * FROM blob_meta_data JOIN blobs ON blob_meta_data.blob_id = blobs.id;
```

You cannot index or sort the entire text columns due to size. You have to index a portion and search for the first 1000 or so characters.

### Enum

Looks like a string, but is stored as a number

```sql
CREATE TABLE orders (
  id SERIAL,
  size ENUM('x-small', 'small', 'medium', 'large', 'x-large')
);

INSERT INTO orders (size) VALUES ('small');

SELECT size FROM orders;
-- small

SELECT size+0 FROM orders;
-- 2
```

These numbers come from when we originally set up the column type:
```sql
-- String  | Position
-- x-small | 1
-- small   | 2
-- medium  | 3
-- large   | 4
-- x-large | 5
```

Position 0 is reserved for INVALID data.

Weird behaviors:
- When using `ORDER BY`, it sorts by the underlying integer value.
- You can't add another option without changing the schema of the table.

### Dates

There are different DATE data types in MySQL:

| Type      | Bytes | Min                 | Max                 |
|-----------|-------|---------------------|---------------------|
| DATE      | 3     | 1000-01-01          | 9999-12-31          |
| DATETIME  | 8     | 1000-01-01 00:00:00 | 9999-12-31 23:59:59 |
| TIMESTAMP | 4     | 1970-01-01 00:00:00 | 2038-01-19 03:14:07 |
| YEAR      | 1     | 1907                | 2155                |
| TIME      | 3     | -838:59:59          | 838:59:59           |


First question to ask: Do you need to store time?

If not, just use the DATE data type.
If storing time, you can use DATETIME and TIMESTAMP.
If you need to use dates that pass 2038, then you should use the DATETIME data type.

DATETIMES have no concept of a Timezone, but TIMESTAMPS are converted to UTC time.

```sql
CREATE TABLE timezone_test (
  `timestamp` TIMESTAMP NULL,
  `datetime` DATETIME NULL
);

INSERT INTO timezone_test VALUES ('2025-01-01 00:00:00', '2025-01-01 00:00:00');

SELECT * FROM timezone_test;
-- timestamp           | datetime
-- 2025-01-01 00:00:00 | 2025-01-01 00:00:00

SET SESSION time_zone '+05:00'; -- CST

SELECT * FROM timezone_test;
-- timestamp           | datetime
-- 2025-01-01 05:00:00 | 2025-01-01 00:00:00 <- different timestamp value

-- INSERT same data with new session timezone
INSERT INTO timezone_test VALUES ('2025-01-01 00:00:00', '2025-01-01 00:00:00');
SELECT * FROM timezone_test;
-- timestamp           | datetime
-- 2025-01-01 05:00:00 | 2025-01-01 00:00:00
-- 2025-01-01 00:00:00 | 2025-01-01 00:00:00 <- New insert
```

This is the big difference between TIMESTAMP and DATETIME, where TIMESTAMP tries
to assist by converting times in and out of your timezone.

Usually this isn't a big deal, cause lots of servers are set to `00:00`

Most web frameworks in their ORM layer will set session times zones to UTC.

```sql
CREATE TABLE users (
  `id` SERIAL,
  `crated_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  `updated_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

INSERT INTO users (id) VALUES (1);
INSERT INTO users (id) VALUES (2);
INSERT INTO users (id) VALUES (3);
-- Done at different times will give different time values
```

### JSON
```sql
CREATE TABLE has_json (
  `json` JSON,
)

INSERT INTO has_json VALUES ("{}");
SELECT * FROM has_json;
-- {}

INSERT INTO has_json VALUES ("{key: value}"); -- Throws JSON validation error
INSERT INTO has_json VALUES ("{\"key\": \"value\"}"); -- Passes JSON validation

SELECT `json`->>"$.key" FROM has_json; -- ->> is the JSON UN-QUOATING OPPERATOR

ALTER TABLE has_json ADD INDEX j (`json`); -- Error cause you cannot add index to JSON data
-- Need to generate indexes on a specific json path, not the entire JSON blob

ALTER TABLE has_json ADD INDEX my_index ((`json`->>"$.key"));
-- Here we are adding an index on the key field withing the json.
-- This allows faster lookups on queries that involve this JSON path.
```

The JSON data type is a BINARY column under the hood, but is optimized for JSON.

When should you use a JSON column?
- Don't do it to avoid creating a real schema.
- Storing payloads from 3rd parties or with schemas that are not going to be concrete
- Something that is Schema-less (no schema defined, or very fluid schema)

Make sure to only fetch for this data when you need it due to its size.
Similar strategies can be used like with LONGSTRING and LONGBLOB

### Unexpected types

#### Boolean
```sql
CREATE TABLE fun (
  is_having_fun BOOLEAN, -- keywork for `tinyint(1) DEFAULT NULL`
);

SHOW CREATE TABLE fun;

-- CREATE TABLE `fun` (
--   `is_having_fun` tinyint(1) DEFAULT NULL,
-- ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

Another way that is not recommended to create a boolean value is creatinga 0 length character column. Where null is false, and empty string is true.
```sql
is_having_fun char(0) null
```

This can be confusing to undertsand and shouldn't be used in practice for that reason.

#### Zipcodes
Zipcodes are typically 5 digits long, so you would assume integers would be enough.
However, some zipcodes have leading 0s that would get stripped away when storing.
You could now use a `char(5)`. Unless you want to store the extended values of the zipcode (`-####`).
Then you would want to use a `char(10)` type.
This would help too with the fact that zipcodes are not all digits. An example is a zipcode in NY that is `#####-SHOE`

#### IP Address
Usually in formats like `127.0.0.1`. With three deliminators and three characters long for each of the 4 spots, you may think a `char(15)` is all you can do.
You can turn an IP Address into an integer though
```sql
SELECT INET_ATON('127.0.0.1');
-- 2130706433
```

You can revert this with the NTOA function to get human readable version. You can store this as an integer now for more compactness and be able to perform easier scan, sort, filter actions on the queries you perform.

For IPv6 addresses, you can use the `INET6_A2N` function, however these can't be stored in an integer column, but a `binary(16)` column would hold it.

Knowing your data is super helpful in understanding the best data type you need to use in your schema for gains that you can get.

### Generated Columns

Allows the DB to do the work for us when we need it to.

```sql
CREATE TABLE emails (
  email VARCHAR(255), -- robert@example.com
  domain VARCHAR(155) -- want to fill this with example.com
);
```

Seeding the data can be done wiht a hook, presave, etc. This is brittle with direct edits in the DB. Generated columns can help here.

```sql
SELECT SUBSTRING_INDEX('robert@example.com', '@', -1); -- extracts domain


-- use the above as the domain columns "formula"
CREATE TABLE emails (
  email VARCHAR(255), -- robert@example.com
  domain VARCHAR(155) AS (SELECT SUBSTRING_INDEX(email, '@', -1))
);

INSERT INTO emails (email) VALUES ('robert@example.com');

SELECT * FROM emails;
-- email              | domain
-- robert@example.com | example.com
```

You are not able to mess this up on DB inserts, due to erroring on adding in values on a generated column
```sql
INSERT INTO email (email, domain) VALUES ('robert@example.com', 'asdfasdf');
-- Error on value of domain not being allowed for generated column
```


You can use different functions, like hashing a column (`MD5(email)`), combining columns (`CONCAT(email, 'asdf')`), doing math (`total DOUBLE AS (price * 1.0825)`).

Volitile (functions that will change over time), cannot be used in a generated column. The functions have to be deterministic. No `now()` or random values can be used.

These columns can be virtual or stored columns. Virtual are generated on queries (calculated at runtime), stored are stored/written on the disk.
Designated with `STORED` and `VIRTUAL` keywords.

Virtual is calculated every time it is used (could cause overhead if complicated to calculate)
Stored is calcualted on changes and not on reads.

> NOTE: VIRTUAL is default

Another good use case is pulling out keys from a json object to store as a top level column.

```sql
CREATE TABLE emails (
  `json` JSON,
  `email` VARCHAR(255) AS (`json`->>'$.email')
);
```

This is a good case for STORED generated column so that you are not hitting the JSON at each reference. It won't have to reference the full JSON.
VIRTUAL columns would have to calculate off the JSON every time. This applies to BLOBS or TEXT columns that are large as well.


### Schema Migrations

Simply, a folder full of sql statements.
These are used to help keep track of schema changes that need to be made as your DB needs change.
With VC, developers are able to have a coded history as well to changes that are made.
Typically, there is an "up" and "down" step to each migration, where the "up" has new changes we are wanting to introduce, while "down" reverts those changes.

Some best practices can be (but can be opinionated):
- always have explicit sql statements in order to show the database state alteration.
- avoid using down migrations.
- use version control to keep track of changes
- consider when to run migration (maintenance window, tool to help keep db connection functional during migration, etc.)



## Indexes
### Intro

1. Indexes are entirely different separate data structure.
  - Note, a table is an index
Indexes create a second data structure. It isn't an attribute on the table.

2. It maintains a copy of (part of) of your data

3. It has a pointer back to the row.
  The index data structure points to the row in the table data structure.

#### Creating indexes
Create as many indexes as you NEED. They are the best tool for creating performant queries.

Create as few as you can get away with. This is because there is a copy of the data that is maintained, and when changes are made on original data, the index could also be updated.

You can look at your DATA to get your Schema (Data -> Schema)
You cannot tell from your data what a good index would be. You have to look at your queries instead. (Query -> Index) Indexes are drivien by queries/access patterns of your application.

### B+ Trees

The underlying data structure for mose indexes in mysql.

Say you have this table:
```sql
SELECT * FROM people;

-- id | name    | email               |
-- 1  | Philip  | Philip@example.com  |
-- 2  | Josh    | Josh@example.com    |
-- 3  | Ben     | Ben@example.com     |
-- 4  | Ashton  | Ashton@example.com  |
-- 5  | Maddie  | Maddie@example.com  |
-- 6  | Abigail | Abigail@example.com |
-- 7  | Jakob   | Jakob@example.com   |
-- 8  | Luke    | Luke@example.com    |
-- 9  | David   | David@example.com   |
```

And you want to query on the name `Jakob`. What happens wihtout an index?

From MySQL
> Without an index, MySQL must begin with the first row and then read through the entire table to find the relevant row. The larger the table, the more this will cost. If the column in question has an index on it, MySQL can quickly determine the position to seek to in the middle of the data file without having to look at all the data.

Visual representation of a B tree for above data:
```
                                Josh
                          /               \
                     Ben                            Maddie
              /             \                   /            \
     Ashton                 Jakob             Luke            Philip
    /      \             /        \           /   \          /      \
Abigail -> Ashton -> Ben David -> Jakob -> Josh -> Luke -> Maddie -> Philip
```

This uses a Binary search to find the value that is == to what you are searching for. Less then goes left, greater than or equal goes right. 
You travel to the leaves of the tree, starting at the root node. This changes from O(n) to O(log(n)) for searching actions.

### Primary keys
Primary keys can be declared multiple ways
```sql
CREATE TABLE users (
  id BIGINT PRIMARY KEY,

  -- defining at table level is useful when decaring multiple primary keys in a table
  -- PRIMARY KEY (a, b, ...)
);

SHOW CREATE TABLE users;
-- CREATE TABLE `users` (
--   `id` biging NOT NULL,
--   PRIMARY KEY (`id`) 
-- ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

Primary keys have indexes created for them automatically. These are unique indexes that are created.
```SQL
SHOW INDEXES FROM users;
-- Table | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null  | Index_type | Comment | Index_comment | Visible | Expression
-- users |          0 | PRIMARY  |            1 | id          | A         |           0 |     NULL | NULL   | EMPTY | BTREE      | EMPTY   | EMPTY         | YES     | NULL
```

Primary Keys will be declared not nullable and have a unique index created on the column.

If you are not careful, you could create wasted indexes on table by creating an index on the primary key.
```sql
CREATE TABLE users (
  id BIGINT PRIMARY KEY,
  name VARCHAR(255),

  INDEX(id)
);

SHOW CREATE TABLE users;
/* 
 * CREATE TABLE `users` (   
 *   `id` biging NOT NULL,
 *   `name` VARCHAR(255) DEFAULT NULL,
 *   PRIMARY KEY (id), -- Two different keys set
 *   KEY `id` (`id`)
 * ); */

-- Duplicated indexwa now exist.
SHOW INDEXES FROM users;
-- Table | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null  | Index_type | Comment | Index_comment | Visible | Expression
-- users |          0 | PRIMARY  |            1 | id          | A         |           0 |     NULL | NULL   | EMPTY | BTREE      | EMPTY   | EMPTY         | YES     | NULL
-- users |          1 | id       |            1 | id          | A         |           0 |     NULL | NULL   | EMPTY | BTREE      | EMPTY   | EMPTY         | YES     | NULL
```

For our integer primary keys, we should make sure we are using the unsigned instance of the numeric type, so we are not wasting any space.
The following example will ensure our primary key will auto increment and will have more space by not needing to utilize the negative values.
```sql
CREATE TABLE users (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255),
);
```

Leaf nodes for a Primary Key hold the data of the table. It holds the whole row data. This means that a primary key lookup is very fast cause once it gets to the correct leaf, all the data in the row is accessible there. The table is an index (clusterd index).

If you don't create a Primary Key when you create your table, mysql will create one for you and handle it in secret. This is because they are very important.

### Secondary keys
A secondary key/index that is not the primary key. You can have as many secondary keys as you want.

The primary and secondary keys are related to each other.

How does the secondary index get the rest of the data when querying for the information?

The secondary index has a pointer to the primary index, where it does a second lookup to get to the primary key leaf node with all the data.

When we look up a value with a secondary index, it does two lookups. One to get the primary key, the other to get the rest of the data.

Every leaf node in the secondary key has the primary key indexed.

### Primary key data types

What data type should you pick?

There can be a risk of using a very compact primary key is running out of room for inserting new data.

You can use unsigned big integers to get a functionally infinite amount of room.

UUIDs and GUIDs are also options.

Primary keys are copied to every secondary index, so that it knows where to go to get all the data. Using strings can make your secondary indexes bigger. And inserts can be hard, cause using strings could cause an insert in the middle of the tree, then the entire tree needs to be re-balanced at that point, which can be costly.

You could use a sorted uuid and guid (possibly ulid) to help reduce this issue.

The argument
> We shouldn't use ids cause we could have an auto-increment attack where someone just changes the number to see what they can get data on. We should always use UUID and GUID.

This is not a good reason. You can use a schema called nano-id that is used in the url and api. This helps with obsfucation. This is an additional column with an index on it.

### Where to add indexes
The query should drive the index.

We start with our access patterns to see where our indexes should be. Sometimes we need to re-write queries, or create new indexes based on how the way we access our data changes.

We should NOT use an index of every column, cause those are duplicate data in different data structures.

Also, we shouldn't just index things that are all in the WHERE query. But that isn't all we need to consider.

Sorting, grouping, joining all play a part as well as the where clause.

```sql
select * from people;

show create table people;

-- CREATE TABLE `people` (
--   `id` bigint unsigned NOT NULL AUTO_INCREMENT,
--   `first_name` VARCHAR(50),
--   `last_name` VARCHAR(50),
--   `state` CHAR(2),
--   `email` VARCHAR(255),
--   `birthday` date,
--   PRIMARY KEY (`id`)
-- );

alter table `people` add index(`birthday`);
-- Where does this query help us?
select * from people where birthday = '1989-02-14';
select * from people order by birthday limit 10;
select count(*) from people group by birthday;

explain select * from people where birthday='1989-02-14';
-- Will show possible keys and the key that will be used
-- if no `key` is shown in the explain, then it is not utilizing an index.

```

#### When to add indexes:
- Do not assume that anything that shows up in the where clause of a query should have an index. Consider all queries being run and their respective access patterns.
- Do not create an index on every column. This will slow down inserts by functionally duplicating your table. It also won't help reads as much as you'd hope.
- Do consider the entire query when deciding which columns to index. This includes sorting, grouping, and joining.
- Do not worry about trying to create the perfect index for every query. It may not always be possible, and sometimes you will have to rework the queries to take advantage of existing indexes.

#### What queries do indexes help with?
Given the following index on the birthday column `alter table people add index birthday_index (birthday);`:
- Checking equality `select * from people where birthday = '1989-02-14';`
- Checking unbound and bound ranges `select * from people where birthday >= '2006-01-01';` and `select * from people where birthday between '2006-01-01' and '2006-12-31';`
- Sorting `select * from people order by birthday limit 10;`
- Grouping `select birthday, count(*) from people group by birthday;`

### Index selectivity
#### Terms
**Cardinality** refers to the number of distinct values in a particular column that an index covers. When you use the SHOW INDEXES command in MySQL, the cardinality column in the output shows you the approximate total number of unique values in a given index column.

**Selectivity**, on the other hand, refers to how unique the values in a column are. It is a measure of how selective an index can be in narrowing down results when queried. The higher the selectivity of an index, the better it is for optimizing query performance.


Thoughts when setting an index
- Which columns are the most selective
  - Think in comparison to primary key
- Will I be using this column alot
- Is there enough distinction in the column
  - Ex. `types` on a `user` table where majority of users are type `user` and not type `admin` might not make sense for an index.
  -  This is not highly selective for `user` values, but could be useful when searching `admin` values.

We can always check the selectivity of the index by doing the following query: `select count(distinct birthday) / count(*) from people` gives us a ratio for seeing which column has a higher selectivity.