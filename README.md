# TiDB Flink 实时数仓

## 使用方法

1. 拉起 docker-compose 集群

```bash
# 克隆项目
git clone https://github.com/LittleFall/flink-tidb-rdw && cd ./flink-tidb-rdw/

# 重置环境
docker-compose down -v; rm -rf logs

# 启动集群
docker-compose up -d
```

在集群启动后，可以通过 localhost:8081 查看 flink dashboard。

2. 注册任务环境

在 MySQL 和 TiDB 中注册好表

```bash
docker-compose exec mysql-server mysql -uroot

DROP DATABASE IF EXISTS test;
CREATE DATABASE test; 
USE test;

create table base (
  base_id int primary key,
  base_location varchar(20)
);
create table stuff(
  stuff_id int primary key,
  stuff_base_id int,
  stuff_name varchar(20)
);
```

```bash
docker-compose exec mysql-server mysql -htidb -uroot -P4000

use test;
create table wide_stuff(
    stuff_id int primary key,
    base_id int,
    base_location varchar(20),
    stuff_name varchar(20)
);
```


3. 通过 Flink SQL Client 编写作业

```bash
docker-compose exec jobmanager ./bin/sql-client.sh embedded -l ./connector-lib
```

注册 Flink 表

```sql
create table base (
    base_id int primary key,
    base_location varchar(20)
) WITH (
    'connector' = 'mysql-cdc',
    'hostname' = 'mysql-server',
    'port' = '3306',
    'username' = 'root',
    'password' = '',
    'database-name' = 'test',
    'table-name' = 'base'
);


create table stuff(
    stuff_id int primary key,
    stuff_base_id int,
    stuff_name varchar(20)
) WITH (
    'connector' = 'mysql-cdc',
    'hostname' = 'mysql-server',
    'port' = '3306',
    'username' = 'root',
    'password' = '',
    'database-name' = 'test',
    'table-name' = 'stuff'
); 

create table wide_stuff(
    stuff_id int primary key,
    base_id int,
    base_location varchar(20),
    stuff_name varchar(20)
) WITH (
	'connector'  = 'jdbc',
    'driver'     = 'com.mysql.cj.jdbc.Driver',
    'url'        = 'jdbc:mysql://tidb:4000/test?rewriteBatchedStatements=true',
    'table-name' = 'wide_stuff',
    'username'   = 'root',
    'password'   = ''
);

create table print_base WITH ('connector' = 'print') LIKE base (EXCLUDING ALL);

create table print_stuff WITH ('connector' = 'print') LIKE stuff (EXCLUDING ALL);

create table print_wide_stuff WITH ('connector' = 'print') LIKE wide_stuff (EXCLUDING ALL);
```

提交作业到 Flink 集群

```sql
insert into wide_stuff
select stuff.stuff_id, base.base_id, base.base_location, stuff.stuff_name
from stuff inner join base
on stuff.stuff_base_id = base.base_id;

insert into print_base select * from base;

insert into print_stuff select * from stuff;

insert into print_wide_stuff
select stuff.stuff_id, base.base_id, base.base_location, stuff.stuff_name
from stuff inner join base
on stuff.stuff_base_id = base.base_id;
```

此时可以在 localhost:8081 看到一共注册了 4 个任务，分别是：
- 将 mysql 中 base 表的修改打印在标准输出中。
- 将 mysql 中 stuff 表的修改打印标准输出中。
- 将 base 表和 stuff 表 join 成 wide_stuff 表，将 wide_stuff 表的修改打印在标准输出中。
- 将 wide_stuff 表的修改写入到 tidb 中。

标准输出可以在本目录下执行 `docker-compose logs -f taskmanager` 持续查看。

4. 在 MySQL 中写入数据，进行测试

```bash
docker-compose exec mysql-server mysql -uroot

use test;
insert into base values (1, 'bj');
insert into stuff values (1, 1, 'zhangsan');
insert into stuff values (2, 1, 'lisi');
insert into base values (2, 'sh');
insert into stuff values (3, 2, 'wangliu');
update stuff set stuff_name = 'wangwu' where stuff_id = 3;
delete from stuff where stuff_name = 'lisi';
```

此时可以在标准输出中看到对应的变化：

```bash
taskmanager_1   | +I(1,bj)
taskmanager_1   | +I(1,1,zhangsan)
taskmanager_1   | +I(2,sh)
taskmanager_1   | +I(2,1,lisi)
taskmanager_1   | +I(3,2,wangliu)
taskmanager_1   | -U(3,2,wangliu)
taskmanager_1   | +U(3,2,wangwu)
taskmanager_1   | +I(1,1,bj,zhangsan)
taskmanager_1   | +I(2,1,bj,lisi)
taskmanager_1   | +I(3,2,sh,wangliu)
taskmanager_1   | -U(3,2,sh,wangliu)
taskmanager_1   | +U(3,2,sh,wangwu)
taskmanager_1   | -D(2,1,lisi)
taskmanager_1   | -D(2,1,bj,lisi)
```

也可以在 TiDB 中查看结果：

```bash
docker-compose exec mysql-server mysql -htidb -uroot -P4000 -e"select * from test.wide_stuff";

+----------+---------+---------------+------------+
| stuff_id | base_id | base_location | stuff_name |
+----------+---------+---------------+------------+
|        1 |       1 | bj            | zhangsan   |
|        3 |       2 | sh            | wangwu     |
+----------+---------+---------------+------------+
```

## 注意点

1. Flink 需要内存较大，请将 docker-compose 集群可用的内存调大，建议 6G 及以上。
2. Flink SQL Client 设计为交互式执行，目前不支持一次输入多条语句，一个可用的替代方案是 apache zeppelin。
3. 可以使用如下命令测试 Kafka 是否接收到了数据

```bash
docker-compose exec mysql-server mysql -uroot -e"insert into test.base values (1, 'bj')";
docker-compose exec kafka /opt/kafka/bin/kafka-topics.sh --list --zookeeper zookeeper:2181  
docker-compose exec kafka /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server kafka:9092 --topic test.base --from-beginning

docker-compose down -v && rm -rf ./logs && find ./config/canal-config -name "meta.dat"|xargs rm -f && docker-compose up -d
```

TODO: 增加一些例子，比如 mysql 中异步读取维表、使用 flink 进行数据异构。
