# MySQL Database Partitioning:
------
> 关于zabbix和MySQL分区表 - 支持zabbix 2.0和2.2，mysql在有外键的表不支持分区表。在zabbix 2.0和2.2中history和trend表没有使用外键，因此是可以在这些表中做分区的。

## Index changes
 1. 如果zabbix的数据库已经有了数据，更改索引可能需要一些时间，根据具体的数据量，需要的时间长短也不一样。
 2. 在某些版本的MySQL索引的改变会使整个表上读锁。貌似mysql 5.6没有这个限制。
所述第一步骤是修改几个索引以允许做分区，按照下面的命令：
```mysql
mysql> Alter table history_text drop primary key, add index (id), drop index history_text_2, add index history_text_2 (itemid, id);
Query OK, 0 rows affected (0.49 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> Alter table history_log drop primary key, add index (id), drop index history_log_2, add index history_log_2 (itemid, id);
Query OK, 0 rows affected (2.71 sec)
Records: 0  Duplicates: 0  Warnings: 0
```
## Stored Procedures
下面开始填写存储过程，需要执行下面的几个存储过程语句，只要能看到"Query OK, 0 rows affected (0.00 sec)"只能就没有什么问题了。
```mysql
DELIMITER $$
CREATE PROCEDURE `partition_create`(SCHEMANAME VARCHAR(64), TABLENAME VARCHAR(64), PARTITIONNAME VARCHAR(64), CLOCK INT)
BEGIN
        /*
           SCHEMANAME = The DB schema in which to make changes
           TABLENAME = The table with partitions to potentially delete
           PARTITIONNAME = The name of the partition to create
        */
        /*
           Verify that the partition does not already exist
        */
 
        DECLARE RETROWS INT;
        SELECT COUNT(1) INTO RETROWS
        FROM information_schema.partitions
        WHERE table_schema = SCHEMANAME AND TABLE_NAME = TABLENAME AND partition_description >= CLOCK;
 
        IF RETROWS = 0 THEN
                /*
                   1. Print a message indicating that a partition was created.
                   2. Create the SQL to create the partition.
                   3. Execute the SQL from #2.
                */
                SELECT CONCAT( "partition_create(", SCHEMANAME, ",", TABLENAME, ",", PARTITIONNAME, ",", CLOCK, ")" ) AS msg;
                SET @SQL = CONCAT( 'ALTER TABLE ', SCHEMANAME, '.', TABLENAME, ' ADD PARTITION (PARTITION ', PARTITIONNAME, ' VALUES LESS THAN (', CLOCK, '));' );
                PREPARE STMT FROM @SQL;
                EXECUTE STMT;
                DEALLOCATE PREPARE STMT;
        END IF;
END$$
DELIMITER ;

DELIMITER $$
CREATE PROCEDURE `partition_drop`(SCHEMANAME VARCHAR(64), TABLENAME VARCHAR(64), DELETE_BELOW_PARTITION_DATE BIGINT)
BEGIN
        /*
           SCHEMANAME = The DB schema in which to make changes
           TABLENAME = The table with partitions to potentially delete
           DELETE_BELOW_PARTITION_DATE = Delete any partitions with names that are dates older than this one (yyyy-mm-dd)
        */
        DECLARE done INT DEFAULT FALSE;
        DECLARE drop_part_name VARCHAR(16);
 
        /*
           Get a list of all the partitions that are older than the date
           in DELETE_BELOW_PARTITION_DATE.  All partitions are prefixed with
           a "p", so use SUBSTRING TO get rid of that character.
        */
        DECLARE myCursor CURSOR FOR
                SELECT partition_name
                FROM information_schema.partitions
                WHERE table_schema = SCHEMANAME AND TABLE_NAME = TABLENAME AND CAST(SUBSTRING(partition_name FROM 2) AS UNSIGNED) < DELETE_BELOW_PARTITION_DATE;
        DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
 
        /*
           Create the basics for when we need to drop the partition.  Also, create
           @drop_partitions to hold a comma-delimited list of all partitions that
           should be deleted.
        */
        SET @alter_header = CONCAT("ALTER TABLE ", SCHEMANAME, ".", TABLENAME, " DROP PARTITION ");
        SET @drop_partitions = "";
 
        /*
           Start looping through all the partitions that are too old.
        */
        OPEN myCursor;
        read_loop: LOOP
                FETCH myCursor INTO drop_part_name;
                IF done THEN
                        LEAVE read_loop;
                END IF;
                SET @drop_partitions = IF(@drop_partitions = "", drop_part_name, CONCAT(@drop_partitions, ",", drop_part_name));
        END LOOP;
        IF @drop_partitions != "" THEN
                /*
                   1. Build the SQL to drop all the necessary partitions.
                   2. Run the SQL to drop the partitions.
                   3. Print out the table partitions that were deleted.
                */
                SET @full_sql = CONCAT(@alter_header, @drop_partitions, ";");
                PREPARE STMT FROM @full_sql;
                EXECUTE STMT;
                DEALLOCATE PREPARE STMT;
 
                SELECT CONCAT(SCHEMANAME, ".", TABLENAME) AS `table`, @drop_partitions AS `partitions_deleted`;
        ELSE
                /*
                   No partitions are being deleted, so print out "N/A" (Not applicable) to indicate
                   that no changes were made.
                */
                SELECT CONCAT(SCHEMANAME, ".", TABLENAME) AS `table`, "N/A" AS `partitions_deleted`;
        END IF;
END$$
DELIMITER ;

DELIMITER $$
CREATE PROCEDURE `partition_maintenance`(SCHEMA_NAME VARCHAR(32), TABLE_NAME VARCHAR(32), KEEP_DATA_DAYS INT, HOURLY_INTERVAL INT, CREATE_NEXT_INTERVALS INT)
BEGIN
        DECLARE OLDER_THAN_PARTITION_DATE VARCHAR(16);
        DECLARE PARTITION_NAME VARCHAR(16);
        DECLARE LESS_THAN_TIMESTAMP INT;
        DECLARE CUR_TIME INT;
 
        CALL partition_verify(SCHEMA_NAME, TABLE_NAME, HOURLY_INTERVAL);
        SET CUR_TIME = UNIX_TIMESTAMP(DATE_FORMAT(NOW(), '%Y-%m-%d 00:00:00'));
 
        SET @__interval = 1;
        create_loop: LOOP
                IF @__interval > CREATE_NEXT_INTERVALS THEN
                        LEAVE create_loop;
                END IF;
 
                SET LESS_THAN_TIMESTAMP = CUR_TIME + (HOURLY_INTERVAL * @__interval * 3600);
                SET PARTITION_NAME = FROM_UNIXTIME(CUR_TIME + HOURLY_INTERVAL * (@__interval - 1) * 3600, 'p%Y%m%d%H00');
                CALL partition_create(SCHEMA_NAME, TABLE_NAME, PARTITION_NAME, LESS_THAN_TIMESTAMP);
                SET @__interval=@__interval+1;
        END LOOP;
 
        SET OLDER_THAN_PARTITION_DATE=DATE_FORMAT(DATE_SUB(NOW(), INTERVAL KEEP_DATA_DAYS DAY), '%Y%m%d0000');
        CALL partition_drop(SCHEMA_NAME, TABLE_NAME, OLDER_THAN_PARTITION_DATE);
 
END$$
DELIMITER ;

DELIMITER $$
CREATE PROCEDURE `partition_verify`(SCHEMANAME VARCHAR(64), TABLENAME VARCHAR(64), HOURLYINTERVAL INT(11))
BEGIN
        DECLARE PARTITION_NAME VARCHAR(16);
        DECLARE RETROWS INT(11);
        DECLARE FUTURE_TIMESTAMP TIMESTAMP;
 
        /*
         * Check if any partitions exist for the given SCHEMANAME.TABLENAME.
         */
        SELECT COUNT(1) INTO RETROWS
        FROM information_schema.partitions
        WHERE table_schema = SCHEMANAME AND TABLE_NAME = TABLENAME AND partition_name IS NULL;
 
        /*
         * If partitions do not exist, go ahead and partition the table
         */
        IF RETROWS = 1 THEN
                /*
                 * Take the current date at 00:00:00 and add HOURLYINTERVAL to it.  This is the timestamp below which we will store values.
                 * We begin partitioning based on the beginning of a day.  This is because we don't want to generate a random partition
                 * that won't necessarily fall in line with the desired partition naming (ie: if the hour interval is 24 hours, we could
                 * end up creating a partition now named "p201403270600" when all other partitions will be like "p201403280000").
                 */
                SET FUTURE_TIMESTAMP = TIMESTAMPADD(HOUR, HOURLYINTERVAL, CONCAT(CURDATE(), " ", '00:00:00'));
                SET PARTITION_NAME = DATE_FORMAT(CURDATE(), 'p%Y%m%d%H00');
 
                -- Create the partitioning query
                SET @__PARTITION_SQL = CONCAT("ALTER TABLE ", SCHEMANAME, ".", TABLENAME, " PARTITION BY RANGE(`clock`)");
                SET @__PARTITION_SQL = CONCAT(@__PARTITION_SQL, "(PARTITION ", PARTITION_NAME, " VALUES LESS THAN (", UNIX_TIMESTAMP(FUTURE_TIMESTAMP), "));");
 
                -- Run the partitioning query
                PREPARE STMT FROM @__PARTITION_SQL;
                EXECUTE STMT;
                DEALLOCATE PREPARE STMT;
        END IF;
END$$
DELIMITER ;
```

## Using the stored procedures
存储过程执行完成之后，现在是要如何使用存储过程了，使用类似下面的SQL：
```mysql
CALL partition_maintenance('<zabbix_db_name>', '<table_name>', <days_to_keep_data>, <hourly_interval>, <num_future_intervals_to_create>)
```
### 这有几点需要注意：
	1. 上面的存储过程使用了变量，所以你需要提供zabbix数据库的库名，表名作为参数。
	2. 这个存储过程的变化依赖时间间隔，以小时为时间间隔。
	3. 当你使用时，需要明确分区的时间间隔，比如一天一个分区，时间间隔就是24，一个小时一个分区那么时间间隔就是1.
	4. num_future_intervals_to_create是基于hourly_interval。比如你想创建14天的未来的分区来保存数据。如果你设置的时间间隔是24(一天一个分区)那么num_future_intervals_to_create就是14,如果你设置的时间间隔是1(一个小时一个分区)那么num_future_intervals_to_create就是336(24*14).
	5. 如果设置的时间间隔是一直变化的，只会影响到新的分区，已经存在的分区不会改变。
		1. 如果时间间隔变小，可能得到这样的错误==> "ERROR 1493 (HY000): VALUES LESS THAN value must be strictly increasing for each partition".建议是先删除所有的未来的分区再变更时间间隔。
```mysql
mysql> CALL partition_maintenance('zabbix', 'history', 28, 24, 14);
+-----------------------------------------------------------+
| msg                                                       |
+-----------------------------------------------------------+
| partition_create(zabbix,history,p201404160000,1397718000) |
+-----------------------------------------------------------+
1 row in set (0.39 sec)
。。。。
Query OK, 0 rows affected, 1 warning (2.04 sec)
```

## 提高调用存储过程
可以使用下面的存储过程来提高调用：
```mysql
DELIMITER $$
CREATE PROCEDURE `partition_maintenance_all`(SCHEMA_NAME VARCHAR(32))
BEGIN
                CALL partition_maintenance(SCHEMA_NAME, 'history', 28, 24, 14);
                CALL partition_maintenance(SCHEMA_NAME, 'history_log', 28, 24, 14);
                CALL partition_maintenance(SCHEMA_NAME, 'history_str', 28, 24, 14);
                CALL partition_maintenance(SCHEMA_NAME, 'history_text', 28, 24, 14);
                CALL partition_maintenance(SCHEMA_NAME, 'history_uint', 28, 24, 14);
                CALL partition_maintenance(SCHEMA_NAME, 'trends', 730, 24, 14);
                CALL partition_maintenance(SCHEMA_NAME, 'trends_uint', 730, 24, 14);
END$$
DELIMITER ;
```
上面的会调用partition_maintenance这个存储过程，history和trends表都是按天进行分区，history保存28天数据，trends表保存730天数据。
调用的话就是这样：
```mysql
mysql> CALL partition_maintenance_all('zabbix');
+----------------+--------------------+
| table          | partitions_deleted |
+----------------+--------------------+
| zabbix.history | N/A                |
+----------------+--------------------+
1 row in set (0.01 sec)
....
....
....

+--------------------+--------------------+
| table              | partitions_deleted |
+--------------------+--------------------+
| zabbix.trends_uint | N/A                |
+--------------------+--------------------+
1 row in set (22.85 sec)

Query OK, 0 rows affected, 1 warning (22.85 sec)
```

## Zabbix Housekeeper changes
使用分区表需要关闭zabbix的history/trends的housekeeper。
> * Zabbix 2.0.x：
关闭housekeeper需要变更zabbix_server.conf配置文件：DisableHousekeeping=1.关闭了housekeeper，old events, audit entries, and user sessions也无法删除了。
> * Zabbix 2.2.x：
ZABBIX 2.2引入了更精确的housekeeper。所有的housekeeper配置在zabbix的web端更改，"Administration" -> "General" ，选择"Housekeeping" ，确保history和trends栏的"Enable internal housekeeping"的对勾去掉。

## 最后
不要让这些表超时了你设置的提前创建的分区，比如14天，这些表在未来的14天内是没有问题的，但是到第15天就会出错，所以要在14天前提前创建分区表。比如在crontab里面添加一条,每天14点30分执行一次:
```shell
30 14 * * * mysql zabbix -e "CALL partition_maintenance_all('zabbix');"
```
## 修改分区表存储的个数
1. 查看mysql中已有的存储过程：
```mysql
mysql> show procedure status; 
+--------+---------------------------+-----------+----------------+---------------------+---------------------+---------------+---------+----------------------+----------------------+--------------------+
| Db     | Name                      | Type      | Definer        | Modified            | Created             | Security_type | Comment | character_set_client | collation_connection | Database Collation |
+--------+---------------------------+-----------+----------------+---------------------+---------------------+---------------+---------+----------------------+----------------------+--------------------+
| zabbix | partition_create          | PROCEDURE | root@localhost | 2014-10-20 16:21:01 | 2014-10-20 16:21:01 | DEFINER       |         | utf8                 | utf8_general_ci      | latin1_swedish_ci  |
| zabbix | partition_drop            | PROCEDURE | root@localhost | 2014-10-20 16:21:13 | 2014-10-20 16:21:13 | DEFINER       |         | utf8                 | utf8_general_ci      | latin1_swedish_ci  |
| zabbix | partition_maintenance     | PROCEDURE | root@localhost | 2014-10-20 16:21:20 | 2014-10-20 16:21:20 | DEFINER       |         | utf8                 | utf8_general_ci      | latin1_swedish_ci  |
| zabbix | partition_maintenance_all | PROCEDURE | root@localhost | 2015-09-08 10:42:51 | 2015-09-08 10:42:51 | DEFINER       |         | utf8                 | utf8_general_ci      | latin1_swedish_ci  |
| zabbix | partition_verify          | PROCEDURE | root@localhost | 2014-10-20 16:21:27 | 2014-10-20 16:21:27 | DEFINER       |         | utf8                 | utf8_general_ci      | latin1_swedish_ci  |
+--------+---------------------------+-----------+----------------+---------------------+---------------------+---------------+---------+----------------------+----------------------+--------------------+
5 rows in set (0.00 sec)
```
2. 删掉partition_maintenance_all的存储过程：
```mysql
mysql> drop procedure partition_maintenance_all;
```
3. 修改需要调整的存储过程，然后创建。
```mysql
DELIMITER $$
CREATE PROCEDURE `partition_maintenance_all`(SCHEMA_NAME VARCHAR(32))
BEGIN
                CALL partition_maintenance(SCHEMA_NAME, 'history', 7, 24, 14);
                CALL partition_maintenance(SCHEMA_NAME, 'history_log', 7, 24, 14);
                CALL partition_maintenance(SCHEMA_NAME, 'history_str', 7, 24, 14);
                CALL partition_maintenance(SCHEMA_NAME, 'history_uint', 7, 24, 14);
                CALL partition_maintenance(SCHEMA_NAME, 'history_text', 7, 24, 14);
                CALL partition_maintenance(SCHEMA_NAME, 'trends', 730, 24, 14);
                CALL partition_maintenance(SCHEMA_NAME, 'trends_uint', 730, 24, 14);
END$$
DELIMITER ;
```
