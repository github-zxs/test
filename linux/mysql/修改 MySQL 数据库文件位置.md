### 修改 MySQL 数据库文件位置（Ubuntu 24.04 专属）

MySQL 默认数据库文件位置为 `/var/lib/mysql`，以下步骤将其修改为自定义路径（以 `/data/mysql/data` 为例，可替换为你的目标路径），核心需完成「停止服务→迁移数据→配置权限→修改配置→重启生效」，同时需处理Ubuntu 24.04的AppArmor安全限制，全程需谨慎操作，避免数据丢失。

#### 1. 再次停止 MySQL 服务

修改数据目录前，必须确保服务完全停止，避免数据损坏：

```bash
sudo systemctl stop mysql
```

**注意**：停止服务后，可执行 `sudo systemctl status mysql` 确认服务状态为 `inactive (dead)`，确保完全停止。

#### 2. 安装 rsync 工具（高效迁移数据）

使用 rsync 工具迁移数据，可保留文件属性和权限，避免数据丢失或异常，比普通复制更安全：

```bash
sudo apt update && sudo apt install rsync -y
```

#### 3. 创建自定义数据库目录并设置权限

创建目标目录（替换 `/data/mysql/data` 为你的自定义路径），并赋予MySQL用户完全权限（权限错误会导致服务无法启动）：

```bash
# 创建自定义目录（递归创建父目录，避免目录不存在报错）
sudo mkdir -p /data/mysql/data
# 赋予mysql用户所有权（关键，确保MySQL能读写目录）
sudo chown -R mysql:mysql /data/mysql/data
# 设置目录权限（755为合理权限，避免过高或过低）
sudo chmod -R 755 /data/mysql/data
```

**注意**：自定义路径建议选择空间充足的分区（如挂载的硬盘），避免后续数据增长导致空间不足；若路径包含特殊字符，需用引号包裹。

#### 4. 迁移原有数据库文件到新目录

使用 rsync 同步原有数据到新目录，确保数据完整一致（保留所有文件属性、权限，同步过程可查看进度）：

```bash
sudo rsync -ahX --progress --checksum /var/lib/mysql/ /data/mysql/data/
```

同步完成后，可验证数据一致性（无输出即表示所有文件同步一致，若有输出需重新同步）：

```bash
diff -rq /var/lib/mysql/ /data/mysql/data/
```

**注意**：同步过程中不要中断命令，否则会导致数据不完整；若原有数据量较大，同步时间会相应延长，耐心等待即可。

#### 5. 配置 （Ubuntu 24.04 必做）

Ubuntu 24.04 默认启用 AppArmor 安全模块，会限制MySQL访问自定义目录，需修改其配置文件，允许MySQL访问新的数据目录：

```bash
# 编辑AppArmor配置文件（MySQL专属配置）
sudo nano /etc/apparmor.d/usr.sbin.mysqld
```

打开文件后，找到包含 `/var/lib/mysql/` 的相关配置（通常是一堆允许访问的规则），在其下方添加新目录的访问权限，添加内容如下（替换`/data/mysql/data` 为你的自定义路径）：

```bash
/data/mysql/data/ r,
/data/mysql/data/** rwk,
```

添加完成后，按 `Ctrl+O` 保存，按 `Ctrl+X` 退出编辑器。

重启 AppArmor 服务，使配置生效：

```bash
sudo systemctl restart apparmor
```

**注意**：若未配置AppArmor权限，MySQL启动时会报权限错误，无法正常访问新目录；添加规则时，路径末尾的 `/` 和权限标识（r、rwk）不要遗漏。

#### 6. 修改 MySQL 主配置文件（指定新数据目录）

编辑MySQL的主配置文件，将默认数据目录改为自定义路径，确保MySQL启动时加载新目录的文件：

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

找到 `datadir` 配置项（默认值为 `datadir = /var/lib/mysql`），将其修改为自定义路径：

```bash
datadir = /data/mysql/data
```

同时，确认配置文件中`socket` 项（默认 `socket = /var/run/mysqld/mysqld.sock`）无需修改，保持默认即可。

保存退出（Ctrl+O → 回车 → Ctrl+X）。

#### 7. 备份原有目录（可选，建议操作）

为避免后续出现问题无法回滚，建议将原有默认目录备份（不删除，仅重命名）：

```bash
sudo mv /var/lib/mysql /var/lib/mysql.bak
```

**注意**：备份完成后，不要手动删除 `/var/lib/mysql.bak`，待确认新目录正常使用后，再根据需求删除（释放空间）。

#### 8. 启动 MySQL 服务并验证

启动MySQL服务，检查是否能正常启动：

```bash
sudo systemctl start mysql
```

查看服务状态，确认启动成功（显示 `active (running)`）：

```bash
sudo systemctl status mysql
```

若启动失败，可查看错误日志排查问题：

```bash
sudo tail -n 20 /var/log/mysql/error.log
```

启动成功后，登录MySQL，验证数据是否完整（可查看数据库列表，确认原有数据存在）：

```bash
mysql -u root -p
```

进入MySQL后，执行以下命令查看数据目录是否生效：

```sql
show variables like 'datadir';
```

输出结果中，`Value` 应为你的自定义路径（如 `/data/mysql/data`），即表示修改成功。

#### 8. 常见问题排查

- 启动失败，提示“Permission denied”：检查自定义目录权限，确保 `mysql:mysql` 为所有者，权限为755；同时确认AppArmor配置正确，已重启AppArmor服务。
    
- 启动成功，但数据丢失：确认rsync同步命令正确（末尾有 `/`），重新执行同步命令，再验证数据一致性。
    
- 重启系统后MySQL无法启动：检查 `/data/mysql/data` 目录是否存在，若被清理，重新创建目录并赋予权限，再启动服务（可设置目录自动创建脚本，避免重启后丢失）。
    

**总结**：修改MySQL数据库文件位置的核心是「迁移数据+配置权限+修改配置」，Ubuntu 24.04 需重点处理AppArmor权限，全程确保服务停止、数据同步完整，避免因操作不当导致数据丢失或服务异常。