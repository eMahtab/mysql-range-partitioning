# MySQL Table Partitioning

## Step 1 : Run the MySQL using docker compose

```yml
---
version: "2"
services:
  mysql_db:
    image: mysql:8.0
    container_name: mysql-instance-1
    volumes:
      - mysql-instance-1-volume:/tmp
    command:
      [
        "mysqld",
        "--datadir=/tmp/mysql/data",
        "--log-bin=bin.log",
        "--server-id=1"
      ]
    environment:
      &mysql-default-environment
      MYSQL_ROOT_PASSWORD: toor
      MYSQL_DATABASE: test
    ports:
      - "3308:3306"

volumes:
  mysql-instance-1-volume:
```

```docker
docker compose up
```
!["Run MySQL instance as a docker container"](docker-compose-up.png?raw=true)

!["MySQL instance as a docker container"](docker-container.png?raw=true)

## Step 2 : Dataset generation : Create users and messages table and insert data

The `test` database contains two tables `users` and `messages`. Records were inserted in batches by a Java program.

### Dataset Size :

**users table = 10 Million records**

**messages table = 100 Million records**


#### Schema  : 
**The tables were originally created using below DDL statements :**

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
!["Space used by tables"](docker-volume.png?raw=true)

!["users and messages tables and binary log files size"](database-tables-and-binary-log-files.png?raw=true)

!["messages by date"](messages-by-date.png?raw=true)

## Indexes in users and messages tables:

Below is the output from **`SHOW INDEX FROM users`** and **`SHOW INDEX FROM messages`**

!["users and messages table indexes"](table-indexes.png?raw=true)

As we can see from above screenshot, **`users` table have one index, which is Primary index on `id` column**

And **`messages` table have one Primary index (on column `id`) and two Secondary indexes (one on column `sender_id` and other one on column `recipient_id`)**

If you see the `CREATE TABLE` command above for `messages` table, we did not explicitly created secondary indexes, it was automatically created by MySQL for enforcing referential integrity **efficiently**. Operations like `ON DELETE CASCADE` or `ON UPDATE CASCADE` need to locate and modify rows in the referencing table quickly.

You can always check the indexes in a table using `SHOW INDEX FROM` command, to get more details.

# Step 3 : Preparing for Table partitioning
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

# Step 4 : Partitioning the table

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
