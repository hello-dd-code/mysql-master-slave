# MySQL 主从搭建主从示例

> 此项目用于记录 Docker Compose 方式搭建 MySQL 8.0.34 的主从同步。

## 端口

主库：3306
从库：3307

## 运行

将项目代码克隆到本地以后，进入到工作目录中，执行：

```shell
docker compose up -d
```

## 主库操作

```shell
# 进入到主库容器内
docker exec -it mysql-master /bin/bash

# 连接 MySQL
mysql -u root -p

# 创建同步用户
CREATE USER 'replicator' @'%' IDENTIFIED BY '123123';

# 授权
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';

# 刷新
FLUSH PRIVILEGES;

# 查看主库状态
SHOW MASTER STATUS;
```

在执行了 `SHOW MASTER STATUS;` 以后，应该会得到如下类似的信息：

```mysql
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |     666 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

需要记录下其中 `File` 和 `Position` 两个字段的值，稍后在从库配置上需要用到。  
**查询完成之后不要再操作主库，操作后记录值会变，需要重新获取**

## 从库操作

### 配置主库信息

```mysql
CHANGE MASTER TO MASTER_HOST = 'master',
MASTER_USER = 'replicator',
MASTER_PASSWORD = '123123',
MASTER_LOG_FILE = 'mysql-bin.000003',
MASTER_LOG_POS = 666;
```
#### 命令参数解释

- **`MASTER_HOST`**
  - **解释**: 指定主数据库服务器的主机名或IP地址。该参数告诉从库应该从哪个主机同步数据。
  - **示例**: `MASTER_HOST = 'master'` 表示从库将连接到主机名为 `master` 的主数据库。

- **`MASTER_USER`**
  - **解释**: 指定从库用于连接主数据库的用户名。这个用户需要有适当的权限来执行复制操作。
  - **示例**: `MASTER_USER = 'replicator'` 表示从库将使用 `replicator` 用户名来连接主数据库。

- **`MASTER_PASSWORD`**
  - **解释**: 为连接主数据库的用户提供密码。此密码用于验证 `MASTER_USER`。
  - **示例**: `MASTER_PASSWORD = '123123'` 表示密码为 `123123`。

- **`MASTER_LOG_FILE`**
  - **解释**: 指定主数据库的二进制日志文件名，从这个日志文件开始，读取数据进行同步。
  - **示例**: `MASTER_LOG_FILE = 'mysql-bin.000003'` 表示从库将从名为 `mysql-bin.000003` 的二进制日志文件开始读取数据。

- **`MASTER_LOG_POS`**
  - **解释**: 指定从库在二进制日志文件中的位置，从这个位置开始读取数据。通常与 `MASTER_LOG_FILE` 一起使用，精确定位同步点。
  - **示例**: `MASTER_LOG_POS = 666` 表示从库将从日志文件的第 666 个字节位置开始读取数据。

需要将上面的 `MASTER_LOG_FILE` 和 `MASTER_LOG_POS` 两个字段的值修改为刚刚在主库中看到的值;

### 开启从库

```mysql
START SLAVE;
```

### 查看状态

```mysql
SHOW SLAVE STATUS;
```

在执行 `SHOW SLAVE STATUS;` 以后可以看到以下输出：
```

主要观察以下三项的值都正确，则配置成功，可以去主库进行某些操作看看能否同步到从库：

1. `Slave_IO_State` 的值为 `Waiting for source to send event`。
2. `Slave_IO_Running` 的值为 `Yes`。
3. `Slave_SQL_Running` 的值为 `Yes`。

如果以上三项的值不符合预期，可以通过查看 `Last_Error`、`Last_IO_Error`、`Last_SQL_Error` 等字段查看错误。