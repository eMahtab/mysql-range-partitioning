# MySQL Table Partitioning

**!!! Note :** This demo was done with MySQL version 8.0, but it should work with older or newer versions as well.

# Step 1 : Database Setup

The `test` database contains two tables `users` and `messages`, the tables and records in the tables are created by importing the (https://github.com/eMahtab/mysql-test-dataset/blob/main/users-and-messages/test_database.zip) database dump.

Its medium sized database having in total 110 Million records, so **importing the database dump will take a long time**.

!["database dump import"](data-import-successful.png?raw=true)

### Dataset Size :

**users table = 10 Million records**

**messages table = 100 Million records**

!["users and messages table in test database"](tables.png?raw=true)

#### Schema  : 
**The tables were originally created using below DDL statements :**

`users` table have id (which denotes user id) as Primary Key, and `messages` table also have id (which denotes message id) as Primary Key. 

`messages` table have columns `sender_id` and `recipient_id` which are Foreign Key, referencing `id` column of `users` table

```sql
CREATE TABLE users (
    id BIGINT,
    name VARCHAR(50),
    username VARCHAR(30),
    PRIMARY KEY (id)
);

CREATE TABLE messages (
    id BIGINT,
    sender_id BIGINT,
    recipient_id BIGINT,
    message TEXT,
    created_at DATETIME NOT NULL,
    edited_at DATETIME DEFAULT NULL,
    deleted_at DATETIME DEFAULT NULL,
    PRIMARY KEY (id),
    FOREIGN KEY (sender_id) REFERENCES users(id),
    FOREIGN KEY (recipient_id) REFERENCES users(id)
);
```

!["users and messages table records"](table-records.png?raw=true)

### Number of messages by date
!["Number of messages by date"](messages-by-date.png?raw=true)

## Indexes in users and messages tables:

Below is the output from **`SHOW INDEX FROM users`** and **`SHOW INDEX FROM messages`**

!["users and messages table indexes"](table-indexes.png?raw=true)

As we can see from above screenshot, **`users` table have one index, which is Primary index on `id` column**

And **`messages` table have one Primary index (on column `id`) and two Secondary indexes (one on column `sender_id` and other one on column `recipient_id`)**

If you see the `CREATE TABLE` command above for `messages` table, we did not explicitly created secondary indexes, it was automatically created by MySQL for enforcing referential integrity **efficiently**. Operations like `ON DELETE CASCADE` or `ON UPDATE CASCADE` need to locate and modify rows in the referencing table quickly.

You can always check the indexes in a table using `SHOW INDEX FROM` command, to get more details.

# Step 2 : Preparing for Table partitioning
### Dropping the Foreign key constraint
```sql
ALTER TABLE messages DROP CONSTRAINT messages_ibfk_1;
ALTER TABLE messages DROP CONSTRAINT messages_ibfk_2;
```

### Making created_at column as part of Primary Key

We drop the existing primary key, and create a new primary key by adding `created_at` to `id` column. 

**Note : It took around half an hour to drop the previous index and create the new one.**

```sql
ALTER TABLE messages DROP PRIMARY KEY, ADD PRIMARY KEY (id, created_at);
```
!["adding created_at to primary key"](new-primary-key.png?raw=true)

# Step 3 : Partitioning the table

We partition the messages table using range partition on created_at column based on date.

**Below we create 10 partitions, e.g. all the messages which were sent between 2024-12-04 00:00:00 and 2024-12-04 23:59:59 would go inside partition `pDec04`**

```sql
ALTER TABLE messages PARTITION BY RANGE (TO_DAYS(created_at)) (
   PARTITION pBeforeDec01 VALUES LESS THAN (TO_DAYS('2024-12-01')),
   PARTITION pDec01 VALUES LESS THAN (TO_DAYS('2024-12-02')),
   PARTITION pDec02 VALUES LESS THAN (TO_DAYS('2024-12-03')),
   PARTITION pDec03 VALUES LESS THAN (TO_DAYS('2024-12-04')),
   PARTITION pDec04 VALUES LESS THAN (TO_DAYS('2024-12-05')),
   PARTITION pDec05 VALUES LESS THAN (TO_DAYS('2024-12-06')),
   PARTITION pDec06 VALUES LESS THAN (TO_DAYS('2024-12-07')),
   PARTITION pDec07 VALUES LESS THAN (TO_DAYS('2024-12-08')),
   PARTITION pDec08 VALUES LESS THAN (TO_DAYS('2024-12-09')),
   PARTITION pAfterDec08 VALUES LESS THAN (MAXVALUE)
);
```
!["Partition the table using range partition"](alter-table-range-partition.png?raw=true)

**Note : It took 7:10 hours (a long time) to partition the original messages table of table data size around 13.2 GB to partition in 10 different partitions.**

!["Messages table partitioned"](messages-table-partitioned.png?raw=true)

```sql
SELECT TABLE_SCHEMA, TABLE_NAME, PARTITION_NAME, 
    (DATA_LENGTH + INDEX_LENGTH) / (1024 * 1024) AS PARTITION_SIZE_MB
FROM information_schema.partitions
WHERE TABLE_NAME = 'messages';
```
!["All 10 partitions"](all-10-partitions.png?raw=true)

```sql
mysql> select count(*) from messages partition (pBeforeDec01);
+----------+
| count(*) |
+----------+
|        0 |
+----------+
1 row in set (0.00 sec)

mysql> select count(*) from messages partition (pDec01);
+----------+
| count(*) |
+----------+
|  7926924 |
+----------+
1 row in set (1.73 sec)

mysql> select count(*) from messages partition (pDec02);
+----------+
| count(*) |
+----------+
| 14290662 |
+----------+
1 row in set (9.44 sec)

mysql> select count(*) from messages partition (pDec03);
+----------+
| count(*) |
+----------+
| 14278081 |
+----------+
1 row in set (13.47 sec)

mysql> select count(*) from messages partition (pDec04);
+----------+
| count(*) |
+----------+
| 14284870 |
+----------+
1 row in set (9.63 sec)

mysql> select count(*) from messages partition (pDec05);
+----------+
| count(*) |
+----------+
| 14286101 |
+----------+
1 row in set (9.45 sec)

mysql> select count(*) from messages partition (pDec06);
+----------+
| count(*) |
+----------+
| 14287221 |
+----------+
1 row in set (9.61 sec)

mysql> select count(*) from messages partition (pDec07);
+----------+
| count(*) |
+----------+
| 14286457 |
+----------+
1 row in set (9.63 sec)

mysql> select count(*) from messages partition (pDec08);
+----------+
| count(*) |
+----------+
|  6359684 |
+----------+
1 row in set (3.85 sec)

mysql> select count(*) from messages partition (pAfterDec08);
+----------+
| count(*) |
+----------+
|        0 |
+----------+
1 row in set (0.00 sec)
```

# Step 4 : Trying different queries

### A query which requires, querying a single partition e.g. pDec06
```sql
select count(*)
from messages
where created_at BETWEEN '2024-12-06 00:00:00' AND '2024-12-06 23:59:59';
```

!["Query result from just one partition"](just-one-partition.png?raw=true)

### A query which requires, querying multiple partition e.g. pDec03, pDec04, pDec05, pDec06
```sql
select *
from messages
where created_at BETWEEN '2024-12-03 00:00:00' AND '2024-12-06 23:59:59' AND sender_id = 73445;
```

!["Querying multiple partitions"](querying-multiple-partitions.png?raw=true)

### A query which requires, querying all partitions
```sql
select count(*)
from messages
```

!["Queries all partitions"](queries-all-partitions.png?raw=true)

### A query which requires, querying all partitions
Even though below query requires checking all the partitions, but since we have secondary index on `sender_id` column, the below query takes very less time to complete.
But if we didn't had index on `sender_id` it would have taken a very long time.

```sql
select count(*)
from messages
where sender_id = 8829;
```

!["Queries all partitions"](queries-all-partitions-2.png?raw=true)



