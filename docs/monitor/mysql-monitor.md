### MYSQL 主从模式监控<Badge type="tip" text="mysql" />

#### 搭建mysql 主从环境

##### 1. docker compose 部署

```yaml
version: "3"

services:
  mysql-master:
    image: docker.m.daocloud.io/library/mysql:8.0.40
    container_name: mysql-master
    command: --server-id=1 --log-bin=mysql-bin --binlog-format=row
    environment:
      MYSQL_ROOT_PASSWORD: qiaoke
    ports:
      - "3307:3306"

  
  mysql-slave:
    image: docker.m.daocloud.io/library/mysql:8.0.40
    container_name: mysql-slave
    depends_on:
      - mysql-master
    command: --server-id=2 --log-bin=mysql-bin --binlog-format=row
    environment:
      MYSQL_ROOT_PASSWORD: qiaoke
    ports:
      - "3308:3306"
```

##### 2. 配置主从复制

主节点配置

```bash
CREATE USER 'replication_user'@'%' IDENTIFIED BY 'qiaoke';
GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'%';
ALTER USER 'replication_user'@'%' IDENTIFIED WITH mysql_native_password BY 'qiaoke';
FLUSH PRIVILEGES;

FLUSH TABLES WITH READ LOCK;  -- 锁定表以确保数据一致性
SHOW MASTER STATUS;            -- 查看当前日志文件和位置
```

从节点配置

```
CHANGE MASTER TO
    MASTER_HOST='mysql-master',
    MASTER_USER='replication_user',
    MASTER_PASSWORD='qiaoke',
    MASTER_LOG_FILE='mysql-bin.000003',  -- 替换为主服务器的日志文件名
    MASTER_LOG_POS=1176;                  -- 替换为主服务器的日志位置
    
START SLAVE;

# 查看复制状态
SHOW SLAVE STATUS\G;
```

#### Zabbix

##### 1. 安装mysql 并配置监控所需账户 `patrol`

```
mysql -u root -p
CREATE USER 'patrol'@'%' IDENTIFIED BY 'patrol';
GRANT REPLICATION CLIENT,PROCESS,SHOW DATABASES,SHOW VIEW ON *.* TO 'patrol'@'%';
quit
```

##### 2. zabbix 前端操作

> https://www.zabbix.com/documentation/current/en/manual/guides/monitor_mysql
>
> https://blog.csdn.net/Eternity_zzh/article/details/126756948

###### 2.1 创建主机

![image-20241102024533045](/image-20241102024533045.png)

###### 2.2 配置宏变量

![image-20241102024559943](/image-20241102024559943.png)

###### 2.3 自定义从库延迟指标

添加自定义监控脚本

```bash
#!/bin/bash

# 获取从库延迟时间
res=$(mysql -h 127.0.0.1 -P 3308 -upatrol -ppatrol -e "show slave status\G" 2> /dev/null |awk '/Seconds_Behind_Master/ {print $2}')

echo $res
```

添加自定义监控配置

```bash
vim /etc/zabbix/zabbix_agent2.d/plugins.d/userparameter_mysql.conf

UserParameter=check_mysqlyc,/bin/bash /etc/zabbix/scripts/mysql.sh
```

zabbix 平台添加监控项

![image-20241102034806471](/image-20241102034806471.png)

##### 4. 配置告警

zabbix 默认模板包含相应告警数据

##### 5. 接入grafana

grafana 安装zabbix插件

![image-20241102042339235](/image-20241102042339235.png)

添加zabbix数据源配置参数

![image-20241102042429594](/image-20241102042429594.png)

![image-20241102042443704](/image-20241102042443704.png)

查看zabbix数据

![image-20241102042600834](/image-20241102042600834.png)

#### Prometheus

> https://github.com/prometheus/mysqld_exporter

##### 1. 安装mysql exporter

```bash
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.15.1/mysqld_exporter-0.15.1.linux-amd64.tar.gz

tar -zxvf mysqld_exporter-0.15.1.linux-amd64.tar.gz
```

##### 2. 配置相关参数

创建监控账户

```sql
CREATE USER 'exporter'@'%' IDENTIFIED BY 'exporter123' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';

flush privileges;
```

创建配置文件

```bash
vim .my.cnf
[client]
host=127.0.0.1
port=3307
user=exporter
password=exporter123
```

启动

```bash
 ./mysqld_exporter --config.my-cnf=/root/mysql/mysqld_exporter-0.15.1.linux-amd64/.my.cnf
```

##### 3. 配置告警

mysql 下线告警配置

```yaml
groups:
  - name: mysql_alerts
    rules:
    - alert: MySQLInstanceDown
      expr: mysql_up{instance="localhost:9104"} == 0
      for: 5m
```

##### 4. 接入grafana

填写 `Mysql` 仪表盘ID Mysql仪表盘`7362` 复制状态仪表盘`7371`

![image-20241103204114680](/image-20241103204114680.png)