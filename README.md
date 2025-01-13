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

**float** is 4 bytes
**double** is 8 bytes
