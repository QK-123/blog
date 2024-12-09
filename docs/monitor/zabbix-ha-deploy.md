## 安装zabbix并启用高可用模式<Badge type="tip" text="zabbix" />

#### 架构图

![image-20241027003727213](/image-20241027003727213.png)

#### 主机列表

| 主机名       | IP              | 组件                                   |
| ------------ | --------------- | -------------------------------------- |
| zabbix-ha1   | 192.168.116.150 | nginx mysql zabbix-server zabbix-agent |
| zabbix-ha2   | 192.168.116.151 | nginx mysql zabbix-server zabbix-agent |
| zabbix-proxy | 192.168.116.152 | proxy  sqlite                          |

口令: `qiaoke`

#### 1. 配置mysql 主从模式

安装Mysql 默认是8.0的版本；设置开机自启；启动Mysql

```bash
apt install -y mysql-server
systemctl enable mysql
systemctl start mysql
systemctl status mysql
```

默认安装未设置root密码，登录设置密码

```bash
mysql -uroot -p
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'qiaoke';
flush privileges;
```

主节点配置

```bash
vim /etc/mysql/my.cnf

[mysqld]
server-id = 1          # 主服务器的唯一 ID
log_bin = mysql-bin    # 启用二进制日志
binlog_do_db = your_database_name  # 指定要复制的数据库（可选）
bind-address = 0.0.0.0

# 注释掉 /etc/mysql/mysql.conf.d/mysqld.cnf
bind-address            = 127.0.0.1

sudo systemctl restart mysql


# 创建复制用户
CREATE USER 'replication_user'@'%' IDENTIFIED BY 'zabbix';
GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'%';
ALTER USER 'replication_user'@'%' IDENTIFIED WITH mysql_native_password BY 'zabbix';
FLUSH PRIVILEGES;


# 获取二进制日志文件名和位置
FLUSH TABLES WITH READ LOCK;  -- 锁定表以确保数据一致性
SHOW MASTER STATUS;            -- 查看当前日志文件和位置

mysql-bin.000001 |      1342
```

从节点配置

```
vim /etc/mysql/my.cnf

[mysqld]
server-id = 2          # 从服务器的唯一 ID

sudo systemctl restart mysql

# 配置复制参数
CHANGE MASTER TO
    MASTER_HOST='192.168.116.150',
    MASTER_USER='replication_user',
    MASTER_PASSWORD='zabbix',
    MASTER_LOG_FILE='mysql-bin.000001',  -- 替换为主服务器的日志文件名
    MASTER_LOG_POS=881;                  -- 替换为主服务器的日志位置

START SLAVE;

# 查看复制状态
SHOW SLAVE STATUS\G;
```

创建zabbix 数据库

```
mysql -u root -p "qiaoke"
# 数据库
create database zabbix character set utf8mb4 collate utf8mb4_bin;
create user 'zabbix'@'%' identified by 'zabbix'; 
grant all privileges on zabbix.* to 'zabbix'@'%';
set global log_bin_trust_function_creators = 1;
flush privileges; 
quit

#将zabbix 数据库表 导入数据库
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix

set global log_bin_trust_function_creators = 0;
```

#### 1. zabbix server 高可用配置

配置zabbix 软件源

```bash
wget https://repo.zabbix.com/zabbix/6.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest+ubuntu22.04_all.deb
dpkg -i zabbix-release_latest+ubuntu22.04_all.deb
apt update
```

安装zabbix server, 前端组件，代理

```bash
apt install zabbix-server-mysql zabbix-frontend-php zabbix-nginx-conf zabbix-sql-scripts zabbix-agent
```

配置Zabbix Server

```bash
vim /etc/zabbix/zabbix_server.conf

# 修改以下内容
DBName=zabbix 
DBUser=zabbix
DBPassword=密码
DBHost=192.168.116.150
DBPort=3306

# 启动zabbix-server
systemctl enable zabbix-server 
systemctl restart zabbix-server
systemctl status zabbix-server

# 查看日志是否有报错
tail -f /var/log/zabbix/zabbix_server.log
```

高可用参数配置

```
vim /etc/zabbix/zabbix_server.conf

HANodeName=zabbix-ha1
NodeAddress=192.168.116.150:10051

HANodeName=zabbix-ha2
NodeAddress=192.168.116.151:10051

systemctl restart zabbix-server
```

#### 2. 配置zabbix 前端页面

配置nginx 监听

```bash
vim  /etc/zabbix/nginx.conf 
# 将文件中的这两行注释去掉
listen 8080;
server_name example.com;
```

重启zabbix-server 并配置nginx php-fpm开机自启

```bash
systemctl restart zabbix-server zabbix-agent nginx php8.1-fpm
systemctl enable nginx php8.1-fpm
```

#### 4. 配置Nginx高可用

在两个台节点安装 `keepalived` `nginx`

```bash
apt install -y haproxy keepalived
```

配置zabbix-ha1 的keepalived

```bash
vim /etc/keepalived/keepalived.conf

# 文件内容
global_defs {
    script_user root
    enable_script_security
}

vrrp_script chk_nginx {                   # 定义一个 vrrp_script，用于监控 nginx 进程
    script "/usr/bin/killall -0 nginx"                   # 监控 nginx 进程是否存在
    interval 2                             # 检测间隔（秒）
    weight 10                              # 如果检测失败，权重减少 10
}

vrrp_instance zabbix-ha1  { 
  state MASTER 
  interface ens33
  virtual_router_id 51 
  priority 244 
  advert_int 1 
  authentication {
    auth_type PASS 
    auth_pass 12345
  } 
  track_script { 
    chk_nginx
  }
  virtual_ipaddress {
    192.168.116.100/24
  }
}
```

配置zabbix-ha2 的keepalived

```bash
vim /etc/keepalived/keepalived.conf

# 文件内容
global_defs {
    script_user root
    enable_script_security
}

vrrp_script chk_nginx {                   # 定义一个 vrrp_script，用于监控 nginx 进程
    script "/usr/bin/killall -0 nginx"                   # 监控 nginx 进程是否存在
    interval 2                             # 检测间隔（秒）
    weight 10                              # 如果检测失败，权重减少 10
}

vrrp_instance zabbix-ha2  { 
  state BACKUP 
  interface ens33 
  virtual_router_id 51 
  priority 243 
  advert_int 1 
  authentication {
    auth_type PASS 
    auth_pass 12345
  } 
  track_script { 
    chk_nginx
  }
  virtual_ipaddress {
    192.168.116.100/24
  }
}
```

验证

```bash
# zabbix-ha1 执行
systemctl stop nginx
```

#### 5. 配置zabbix proxy sqlite

配置zabbix proxy 源

```bash
wget https://repo.zabbix.com/zabbix/6.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest+ubuntu22.04_all.deb
dpkg -i zabbix-release_latest+ubuntu22.04_all.deb
apt update

apt install zabbix-proxy-sqlite3
```

修改zabbix proxy 配置文件

```bash
vim /etc/zabbix/zabbix_proxy.conf

Server=192.168.116.150:10051;192.168.116.151:10051
Hostname=zabbix-proxy
DBName=/tmp/zabbix_proxy.db
```

重启验证

```bash
systemctl restart zabbix-proxy
systemctl enable zabbix-proxy
```

zabbix 页面配置proxy

