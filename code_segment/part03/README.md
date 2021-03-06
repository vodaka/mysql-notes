# 存储过程和函数

## Example 1

创建表

```sql
CREATE TABLE `u_user_cid` (
  `user_id` int(11) unsigned NOT NULL COMMENT '用户ID',
  `cid` int(10) unsigned NOT NULL DEFAULT '0',
  PRIMARY KEY (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

创建函数

```sql
CREATE FUNCTION `get_cid`(uid int) RETURNS int(11)
begin

    declare result  int unsigned;
    UPDATE u_user_cid set cid=last_insert_id(cid+1) where user_id=uid;
    set result = last_insert_id();
    return result;
end
```

初始化

```sql
insert into u_user_cid(user_id, cid) values(10001, 0)
```

调用函数

```sql
mysql> select get_cid(10001) as cid;
+------+
| cid  |
+------+
|    1 |
+------+
1 row in set (0.00 sec)

mysql> select get_cid(10001) as cid;
+------+
| cid  |
+------+
|    2 |
+------+
1 row in set (0.01 sec)

mysql> select get_cid(10001) as cid;
+------+
| cid  |
+------+
|    3 |
+------+
1 row in set (0.00 sec)
```

## Example 2

table: seq

```sql
CREATE TABLE `seq` (
  `name` tinyint(3) unsigned NOT NULL,
  `val` bigint(20) unsigned NOT NULL,
  PRIMARY KEY (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

function: `seq()`

```sql
CREATE FUNCTION `seq`(seq_name int) RETURNS bigint(20)
begin
    declare result  bigint unsigned;
    update seq set val=last_insert_id(val+1) where name=seq_name;
    set result = last_insert_id();
    return result;
end
```

function: `get_sharding_id_v2()`

```sql
CREATE FUNCTION `get_sharding_id_v2`(sequence bigint) RETURNS bigint(20)
begin
    declare result   bigint unsigned;
    declare shard_id int;
    declare cur_tid  bigint;
    declare cur_time char(12);
    declare cur_tsi  char(19);

    set shard_id = @@server_id;
    set cur_tid  = @@pseudo_thread_id;

    set cur_time = curtime(3);
    set cur_tsi  = concat(floor(unix_timestamp(concat(curdate(), ' ', left(cur_time, 8)))), right(cur_time, 3));

    set result   = (cur_tsi - 1370016000000) << 23;
    set result   = result | (shard_id % 8192) << 10;
    set result   = result | (sequence % 1024);

    return result;
end
```

execute

```sql
select get_sharding_id_v2(seq(1))
```

## Example 3

procedure: `get_shard_batch`

```sql
CREATE PROCEDURE `get_shard_batch`(IN num INT)
BEGIN
    DECLARE s int;
    set session sql_log_bin = off;
    set s = 0;
    CREATE TEMPORARY TABLE IF NOT EXISTS tb (id bigint) engine = myisam;
    WHILE s < num DO
        insert into tb select get_sharding_id_v2(seq(1));
        set s = s +1;
    END WHILE;
    select * from tb;
    drop table tb;
    set session sql_log_bin = on;
END
```

execute: `CALL get_shard_batch(3)`

```sql
mysql> CALL get_shard_batch(3);
+---------------------+
|                  id |
|---------------------|
| 1381770927792138092 |
| 1381770927800526701 |
| 1381770927800526702 |
+---------------------+
Query OK, 0 rows affected
Time: 0.006s
```

## Example 4

table: nextval

```sql
CREATE TABLE `nextval` (
  `id` varchar(50) CHARACTER SET latin1 NOT NULL DEFAULT '',
  `val` bigint(20) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

function: `get_nextval`

```sql
CREATE FUNCTION `get_nextval`(`seq_name` varchar(100)) RETURNS bigint(20)
BEGIN
    DECLARE cur_val bigint(20);

    SELECT val INTO cur_val FROM nextval WHERE id = seq_name;

    IF cur_val IS NOT NULL THEN
        UPDATE nextval SET val = last_insert_id( val + 3) WHERE id = seq_name;
	    set cur_val = last_insert_id();
	    return cur_val;
    ELSE
    	insert ignore into nextval set id=seq_name,val=1;
	return 1;
    END IF;
END
```

execute: `get_nextval(:num)`

```sql
mysql> select get_nextval(1);
+----------------+
| get_nextval(1) |
+----------------+
|              1 |
+----------------+
1 row in set (0.05 sec)

mysql> select get_nextval(1);
+----------------+
| get_nextval(1) |
+----------------+
|              4 |
+----------------+
1 row in set (0.00 sec)

mysql> select get_nextval(2);
+----------------+
| get_nextval(2) |
+----------------+
|              1 |
+----------------+
1 row in set (0.00 sec)

mysql> select * from nextval;
+----+------+
| id | val  |
+----+------+
| 1  |   4  |
| 2  |   1  |
+----+------+
1 row in set (0.00 sec)
```

## Example 5

table: seq

```sql
CREATE TABLE `seq` (
  `name` tinyint(3) unsigned NOT NULL,
  `val` bigint(20) unsigned NOT NULL,
  PRIMARY KEY (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
```

function: `seq(:seq_name)`

```sql
CREATE FUNCTION `seq`(seq_name int) RETURNS bigint(20)
begin
    declare result  bigint unsigned;
    update seq set val=last_insert_id(val+1) where name=seq_name;
    set result = last_insert_id();
    return result;
end
```

function: `get_dis_id(:sequence)`

```sql
CREATE FUNCTION `get_dis_id`(sequence bigint) RETURNS bigint(20)
begin
    declare result   bigint unsigned;
    declare shard_id int;
    declare cur_tid  bigint;
    declare cur_time char(12);
    declare cur_tsi  char(19);

    set shard_id = @@server_id;
    set cur_tid  = @@pseudo_thread_id;

    set cur_time = curtime(3);
    set cur_tsi  = concat(floor(unix_timestamp(concat(curdate(), ' ', left(cur_time, 8)))), right(cur_time, 3));

    set result   = (cur_tsi - 1451577600000) << 13;
    set result   = result | (shard_id % 16) << 9;
    set result   = result | (sequence % 512);
    set result   = result - 8953815040000;

    return result;
end
```

execute: `get_dis_id(seq(:num))`

```sql
mysql> select get_dis_id(seq(1));
+----------------------+
|   get_dis_id(seq(1)) |
|----------------------|
|      672287222772323 |
+----------------------+
1 row in set
Time: 0.005s

mysql> select get_dis_id(seq(1));
+----------------------+
|   get_dis_id(seq(1)) |
|----------------------|
|      672287229514340 |
+----------------------+
1 row in set
Time: 0.004s
```

procedure: `get_dis_id_batch(:num)`

```sql
CREATE PROCEDURE `get_dis_id_batch`(IN num INT)
BEGIN
    DECLARE s int;

    set s = 0;
    CREATE TEMPORARY TABLE IF NOT EXISTS tb (id bigint) engine = myisam;
    START TRANSACTION;
    WHILE s < num DO
        insert into tb select  get_dis_id(seq(1));
        set s = s +1;
    END WHILE;
    COMMIT;
    select * from tb;
    drop table tb;
END
```

execute: `call get_dis_id_batch(:num)`

```sql
mysql> CALL get_dis_id_batch(3);
+-----------------+
|              id |
|-----------------|
| 672288391402088 |
| 672288391410281 |
| 672288391410282 |
+-----------------+
Query OK, 0 rows affected
Time: 0.005s
```










