### Zabbix snmp 配置介绍<Badge type="tip" text="zabbix" />

SNMP 在很大程度上促进了统一监控所有设备，简单网络管理协议，主要用于网络设备监控。

#### snmp三种版本协议对比 

| 版本   | 使用方式 | 特点            |
| ------ | -------- | --------------- |
| SNMPv1 | OID      | 未加密          |
| SNMPv2 | OID      | 未加密          |
| SNMPv3 | OID      | 加密;用户名密码 |

#### Linux 配置snmp   本次实验使用v3版本

##### 安装snmp所需包

```bash
apt install snmp snmpd libsnmp-dev
```

##### 创建用户

```bash
#net-snmp-create-v3-user [-ro] [-a authpass] [-x privpass] [-X DES|AES] [username]
net-snmp-create-v3-user -ro -a my_authpass -x my_privpass -A SHA -X AES snmpv3user
```

##### 配置smpd服务

```bash
vim /etc/snmp/snmpd.conf
# 添加到尾行
view systemview included .1
```

##### 配置开机自启并开启服务

```bash
systemctl enable snmpd
systemctl start snmpd
```

##### 验证查看主机描述

```bash
snmpwalk -v3 -u snmpv3user -l authPriv -a SHA -A my_authpass -x AES -X my_privpass 127.0.0.1 1.3.6.1.2.1.1.1
```

![image-20241012104140656](/image-20241012104140656.png)

#### zabbix 平台配置主机

##### 挂载模板

![image-20241012131555641](/image-20241012131555641.png)

##### 配置snmpv3接口

![image-20241012131630754](/image-20241012131630754.png)

##### 查看数据

![image-20241012133643710](/image-20241012133643710.png)