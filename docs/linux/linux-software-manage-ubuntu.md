#### Linux 软件管理  ubuntu22.04<Badge type="tip" text="linux" />

##### 1. 软件包创建

##### 2. 包签名

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

###### 获取密钥

APT需要一个或多个GPG公钥来验证软件包的签名。这些公钥通常存储在系统的密钥环中，如 `/etc/apt/trusted.gpg` 或 `/etc/apt/keyrings/` 中。

###### 验证元数据签名

###### 验证软件包签名

##### 3. 软件仓库管理

查看可用的仓库

```bash
cat /etc/apt/sources.list

# 附加仓库
ls /etc/apt/sources.list.d/
```

软件源格式

```bash
deb <协议>://<服务器地址> <发行版> <组件>
```

**协议:** 通常是 http 或 https，表示访问软件源的协议。

**服务器地址:** 软件包存放的服务器地址。

**发行版:** 操作系统的版本，比如 jammy、focal 等。

**组件:** 软件包的类型，比如 main、universe、restricted 等。

###### 如何创建私有仓库

##### 4. 索引更新

apt 从配置的软件源中获取最新的软件包信息，并更新本地索引。

```bash
sudo apt update
```

##### 5. 软件包安装

###### 搜索软件

```bash
sudo apt-cache search <关键词>

apt-cache search  OpenJDK Development Kit
```

###### 查看软件信息

```bash
sudo apt-cache show <软件包名>

apt-cache show openjdk-21-jdk
```

###### 安装

```bash
apt install -y openjdk-21-jdk
```

查看软件安装信息

```
dpkg -L openjdk-21-jdk

dpkg -L mysql-server-8.0
```

##### 6. 软件包配置

##### 7. 软件包升级

查看可升级的软件

```bash
apt list --upgradable
```

查看指定软件是否有新版本

```bash
apt-cache policy apt
```

##### 8. 软件包卸载

查看已安装软件

```bash
apt list --installed
```

卸载软件

```bash
sudo apt remove <软件包名>
```

彻底删除（包括配置文件）

```bash
sudo apt purge <软件包名>
```

##### 9. 清理和维护

删除不再使用的软件包

```bash
sudo apt autoremove
```