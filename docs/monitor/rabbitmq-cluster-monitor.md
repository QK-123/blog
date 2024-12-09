### RabbitMQ集群监控<Badge type="tip" text="监控"/>

#### 部署

|      主机       | 角色      |        端口        |
| :-------------: | --------- | :----------------: |
| 192.168.116.159 | master    | 5672，15672，25672 |
| 192.168.116.160 | worker-01 | 5672，15672，25672 |
| 192.168.116.162 | worker-02 | 5672，15672，25672 |

##### 安装 RabbitMQ

```bash
sudo apt-get install curl gnupg apt-transport-https -y

curl -1sLf "https://keys.openpgp.org/vks/v1/by-fingerprint/0A9AF2115F4687BD29803A206B73A36E6026DFCA" | sudo gpg --dearmor | sudo tee /usr/share/keyrings/com.rabbitmq.team.gpg > /dev/null
curl -1sLf "https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-erlang.E495BB49CC4BBE5B.key" | sudo gpg --dearmor | sudo tee /usr/share/keyrings/rabbitmq.E495BB49CC4BBE5B.gpg > /dev/null
curl -1sLf "https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-server.9F4587F226208342.key" | sudo gpg --dearmor | sudo tee /usr/share/keyrings/rabbitmq.9F4587F226208342.gpg > /dev/null

# cat cloudsmith.rabbitmq-erlang.E495BB49CC4BBE5B.key | sudo gpg --dearmor | sudo tee /usr/share/keyrings/rabbitmq.E495BB49CC4BBE5B.gpg > /dev/null
# cat cloudsmith.rabbitmq-server.9F4587F226208342.key | sudo gpg --dearmor | sudo tee /usr/share/keyrings/rabbitmq.9F4587F226208342.gpg > /dev/null

sudo tee /etc/apt/sources.list.d/rabbitmq.list <<EOF
## Provides modern Erlang/OTP releases
##
deb [arch=amd64 signed-by=/usr/share/keyrings/rabbitmq.E495BB49CC4BBE5B.gpg] https://ppa1.rabbitmq.com/rabbitmq/rabbitmq-erlang/deb/ubuntu jammy main
deb-src [signed-by=/usr/share/keyrings/rabbitmq.E495BB49CC4BBE5B.gpg] https://ppa1.rabbitmq.com/rabbitmq/rabbitmq-erlang/deb/ubuntu jammy main

# another mirror for redundancy
deb [arch=amd64 signed-by=/usr/share/keyrings/rabbitmq.E495BB49CC4BBE5B.gpg] https://ppa2.rabbitmq.com/rabbitmq/rabbitmq-erlang/deb/ubuntu jammy main
deb-src [signed-by=/usr/share/keyrings/rabbitmq.E495BB49CC4BBE5B.gpg] https://ppa2.rabbitmq.com/rabbitmq/rabbitmq-erlang/deb/ubuntu jammy main

## Provides RabbitMQ
##
deb [arch=amd64 signed-by=/usr/share/keyrings/rabbitmq.9F4587F226208342.gpg] https://ppa1.rabbitmq.com/rabbitmq/rabbitmq-server/deb/ubuntu jammy main
deb-src [signed-by=/usr/share/keyrings/rabbitmq.9F4587F226208342.gpg] https://ppa1.rabbitmq.com/rabbitmq/rabbitmq-server/deb/ubuntu jammy main

# another mirror for redundancy
deb [arch=amd64 signed-by=/usr/share/keyrings/rabbitmq.9F4587F226208342.gpg] https://ppa2.rabbitmq.com/rabbitmq/rabbitmq-server/deb/ubuntu jammy main
deb-src [signed-by=/usr/share/keyrings/rabbitmq.9F4587F226208342.gpg] https://ppa2.rabbitmq.com/rabbitmq/rabbitmq-server/deb/ubuntu jammy main
EOF


sudo apt-get update -y


sudo apt-get install -y erlang-base \
                        erlang-asn1 erlang-crypto erlang-eldap erlang-ftp erlang-inets \
                        erlang-mnesia erlang-os-mon erlang-parsetools erlang-public-key \
                        erlang-runtime-tools erlang-snmp erlang-ssl \
                        erlang-syntax-tools erlang-tftp erlang-tools erlang-xmerl


sudo apt-get install rabbitmq-server -y --fix-missing
```

##### 配置主机名映射

```bash
cat <<EOF >> /etc/hosts
192.168.116.159 master
192.168.116.160 worker-01
192.168.116.162 worker-02
EOF
```

复制主节点  `/var/lib/rabbitmq/.erlang.cookie` 到其他节点

```bash
sudo scp /var/lib/rabbitmq/.erlang.cookie root@192.168.116.157:/var/lib/rabbitmq/.erlang.cookie
sudo scp /var/lib/rabbitmq/.erlang.cookie root@192.168.116.158:/var/lib/rabbitmq/.erlang.cookie
chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie

systemctl restart rabbitmq-server
```

##### `worker-01 worker-02`  加入集群

```bash
rabbitmqctl stop_app
rabbitmqctl reset
rabbitmqctl join_cluster rabbit@master
rabbitmqctl start_app
```

##### 查看集群信息

```bash
rabbitmqctl cluster_status

# 创建登录管理页面账户
sudo rabbitmqctl add_user qiaoke qiaoke
sudo rabbitmqctl set_user_tags qiaoke administrator
sudo rabbitmqctl set_permissions -p / qiaoke ".*" ".*" ".*"
```

![image-20241105224745422](/image-20241105224745422.png)

#### RabbitMQ 监控要点

##### 延迟

- 消息处理延迟
- 网络延迟
- 吞吐量延迟

##### 流量

- 消息发布速率
- 消息消费速率
- 队列深度
- 连接数

##### 错误率 

- 队列过载和异常
- 连接和通道错误
- 死信队列

##### 饱和度

- 队列深度饱和度
- CPU，内存，磁盘使用率

#### Zabbix

##### 三台主机安装zabbix-agent2

```bash
sudo wget https://repo.zabbix.com/zabbix/6.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest+ubuntu22.04_all.deb
sudo dpkg -i zabbix-release_latest+ubuntu22.04_all.deb
sudo apt update
sudo apt install -y zabbix-agent2
```

##### 修改配置文件

```bash
sudo vim /etc/zabbix/zabbix_agent2.conf

Server=192.168.116.128
ServerActive=192.168.116.128
Hostname=192.168.116.158  #节点IP

sudo systemctl restart zabbix-agent2.service
```

##### RabbitMQ 创建监控账户

```bash
rabbitmqctl add_user patrol patrol
rabbitmqctl set_permissions  -p / patrol "" "" ".*"
rabbitmqctl set_user_tags patrol monitoring
```

##### Zabbix 页面创建监控

![image-20241105232459671](/image-20241105232459671.png)

##### 配置宏变量

![image-20241105232529048](/image-20241105232529048.png)

#### Prometheus

##### RabbitMQ 开启 prometheus插件

```bash
rabbitmq-plugins enable rabbitmq_prometheus

# 获取指标测试
curl -s localhost:15692/metrics | head -n 3
```

修改 `prometheus.yml` 配置文件

```bash
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
           - 192.168.116.128:9093

rule_files:
  - rules.yml
  - rules/*.yml


scrape_configs:
  - job_name: 'rabbitmq'
    static_configs:
      - targets: ['192.168.116.159:15692','192.168.116.160:15692','192.168.116.162:15692']
    metrics_path: /metrics
    scheme: http
    basic_auth:
      username: patrol
      password: patrol
```

##### 查看效果

![image-20241105233922317](/image-20241105233922317.png)

##### 配置 Grafana 支持 ID `10991`

![image-20241105234141356](/image-20241105234141356.png)

##### 配置告警

```bash
groups:
  - name: rabbitmq-alerts
    rules:
      - alert: RabbitMQInstanceDown
        expr: up{instance=~".*:15692"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "RabbitMQ node {{ $labels.instance }} is down"
          description: "The RabbitMQ node has been down for more than 5 minutes."
```

##### 关闭一个实例，查看告警

![image-20241105235338703](/image-20241105235338703.png)