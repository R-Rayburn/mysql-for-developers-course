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

