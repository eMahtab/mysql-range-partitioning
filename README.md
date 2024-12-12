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
