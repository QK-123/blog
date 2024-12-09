## 配置zabbix监控<Badge type="tip" text="zabbix" />

### zabbix agent监控

安装 配置zabbix agent2 

```bash
apt install zabbix-agent2
```

配置为被动模式

![image-20240724124936921](/image-20240724124936921.png)

```bash
vim /etc/zabbix/zabbix_agent2.conf

# 修改文件内容
Server=127.0.0.1
Hostname=Zabbix server

# 配置开始自启
systemctl enable zabbix-agent2
systemctl restart zabbix-agent2
```

配置为主动模式

![image-20240724125356215](/image-20240724125356215.png)

```bash
vim /etc/zabbix/zabbix_agent2.conf

# 修改文件内容
ServerActive=127.0.0.1

#重启探针
systemctl restart zabbix-agent2
```

### SNMP 监控

![image-20240724132551749](/image-20240724132551749.png)

在Linux 主机上配置snmp监控 ，安装相应的工具

```bash
apt install snmp snmpd libsnmp-dev

#创建snmap用户
net-snmp-create-v3-user -ro -a my_authpass -x my_privpass -A SHA -X AES snmpv3user

# 修改snmap agent配置文件
vim /etc/snmp/snmpd.conf

# 添加到文件
view systemview included .1

# 配置开机自启 并启动
systemctl enable snmpd
systemctl start snmpd
```

在zabbix平台 创建主机项 并配置

创建主机项，挂载模板 `Linux by SNMP` 查看最新数据

![image-20240724134457426](/image-20240724134457426.png)

### zabbix 简单检查

简单检查用常用于远程主机上无agent的数据采集方式。

zabbix 平台创建简单检查监控项，查看ssh服务状态

![image-20240724140801910](/image-20240724140801910.png)

### zabbix trapper

trapper用于接收主机传来的数据，主机上需要安装 zabbix sender

```bash
apt install zabbix-sender
```

zabbix 平台配置监控项

![image-20240724150841264](/image-20240724150841264.png)

在主机发送数据到监控项

```bash
zabbix_sender -z 127.0.0.1 -s "Zabbix server" -k trap -o "trapper 测试"
```

查看最新数据

![image-20240724153613233](/image-20240724153613233.png)

### 可计算监控项和依赖监控项

计算机监控项就是对现存的值，进行二次计算得到的值。

![image-20240725095504458](/image-20240725095504458.png)

点击下方测试获取最新值

![image-20240725095540479](/image-20240725095540479.png)

依赖监控项是对主监控项获取的数据进行处理后的值

![image-20240725101251720](/image-20240725101251720.png)

`Linux by Zabbix agent` 模板中的CPU 使用率的计算采取的就是这种方式

![image-20240725101445785](/image-20240725101445785.png)

对主数据进行预处理

![image-20240725101514349](/image-20240725101514349.png)

### 外部检查监控(自定义监控)

![image-20240725110146503](/image-20240725110146503.png)

外部检查使用过shell脚本或者二进制程序进行数据获取

配置zbbix server 以支持自定义脚本

```bash
# 查看脚本放置目录
cat /etc/zabbix/zabbix_server.conf | grep ExternalScripts=

# 创建测试脚本
vim /usr/lib/zabbix/externalscripts/test_external

# 脚本内容
#!/bin/bash 
echo $1

# 添加执行权限 
chmod +x /usr/lib/zabbix/externalscripts/test_external
chown zabbix:zabbix /usr/lib/zabbix/externalscripts/test_external
```

zabbix 平台配置监控项

![image-20240725135335648](/image-20240725135335648.png)

测试查看数据

![image-20240725135354955](/image-20240725135354955.png)

### 配置HTTP Agent监控

![image-20240725142050824](/image-20240725142050824.png)

zabbix 平台配置监控项

![image-20240725142316719](/image-20240725142316719.png)

可以通过返回值预处理来，获取需要的数据。