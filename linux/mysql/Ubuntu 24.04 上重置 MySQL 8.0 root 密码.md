Ubuntu 24.04 上重置 MySQL 8.0 root 密码，核心是先创建缺失的 `/var/run/mysqld` 目录，再用 `--skip-grant-tables` 免密启动、改密、恢复正常启动。下面是**适配 Ubuntu 24.04、可直接复制执行**的完整步骤。

### 一、停止 MySQL 服务

```bash
sudo systemctl stop mysql
```

### 二、创建缺失的 socket 目录（解决 2002 报错）

Ubuntu 24.04 重启后 `/var/run/mysqld` 会被清理，必须手动重建：

```bash
sudo mkdir -p /var/run/mysqld
sudo chown mysql:mysql /var/run/mysqld
sudo chmod 755 /var/run/mysqld
```

### 三、以免密模式启动 MySQL（跳过权限验证）

```bash
sudo mysqld_safe --skip-grant-tables --skip-networking &
```

- `--skip-grant-tables`：免密登录
    
- `--skip-networking`：仅本地连接，更安全
    
- `&`：后台运行，不占用终端
    

### 四、免密登录并重置密码（MySQL 8.0+ 专用）

新开终端执行：

```bash
mysql -u root
```

进入 MySQL 后执行：

```sql
-- 刷新权限（必须先执行）
FLUSH PRIVILEGES;

-- 重置密码（替换为你的新密码，推荐用 mysql_native_password 兼容 GUI）
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '你的新密码';

-- 再次刷新权限
FLUSH PRIVILEGES;

-- 退出
exit;
```

### 五、恢复正常启动 MySQL

1. 停止安全模式进程
    

```bash
sudo killall -9 mysqld mysqld_safe
```

2. 正常启动服务
    

```bash
sudo systemctl start mysql
```

### 六、验证新密码

```bash
mysql -u root -p
```

输入新密码，能登录即成功。

---

### 常见问题（Ubuntu 24.04 专属）

#### 1. 仍报 `Can't connect to local MySQL server`

- 确认 `/var/run/mysqld` 存在且权限正确
    
- 重新执行：
    

```bash
sudo systemctl stop mysql
sudo mkdir -p /var/run/mysqld
sudo chown mysql:mysql /var/run/mysqld
sudo mysqld_safe --skip-grant-tables --skip-networking &
```

#### 2. 改密报错（如 `Unknown system variable`）

先执行 `FLUSH PRIVILEGES;`，再用兼容方式改密：

```sql
USE mysql;
FLUSH PRIVILEGES;
UPDATE user SET authentication_string = '', plugin = 'mysql_native_password' WHERE User = 'root' AND Host = 'localhost';
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY '你的新密码';
```

#### 3. 重启后无法登录

检查 `/etc/mysql/mysql.conf.d/mysqld.cnf`，确保无 `skip-grant-tables` 残留，然后重启服务。

需要我帮你把这些步骤写成一个**一键重置脚本**，以后忘记密码直接运行即可吗？