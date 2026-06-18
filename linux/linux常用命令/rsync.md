rsync 最实用命令合集（直接复制就能用）

下面这些是运维、开发、备份**最常用**的模板，按场景分类，记熟够用很久。

---

## 一、基础本地同步

### 1. 标准归档同步（最常用）

```bash
rsync -av src/ dest/
```

- `-a` = 归档（递归 + 保留权限、属主、时间、软链接）
    
- `-v` = 显示过程 注意：**src 后面加 / 表示同步目录内容，不加 / 会把目录本身拷过去**
    

### 2. 只同步新增 / 修改，不删目标文件

```bash
rsync -av src/ dest/
```

### 3. 严格镜像（目标多余文件会删除）

```bash
rsync -av --delete src/ dest/
```

⚠️ 慎用，会删除目标端独有的文件

---

## 二、远程传输（服务器 ↔ 本地）

### 1. 本地 → 远程服务器

```bash
rsync -avz local_dir/ user@remote_ip:/path/
```

### 2. 远程服务器 → 本地

```bash
rsync -avz user@remote_ip:/path/ local_dir/
```

### 3. 指定 SSH 端口（非 22 端口）

```bash
rsync -avz -e "ssh -p 2222" src/ user@ip:/dest/
```

参数说明：

- `-z` 传输时压缩，网络慢必加
    
- `-P` = --partial --progress 断点续传 + 显示进度
    

---

## 三、带进度、断点续传（大文件必备）

```bash
rsync -avzP src/ dest/
```

适合：大文件、ISO、虚拟机磁盘、数据库备份

---

## 四、限速传输（避免占满带宽）

```bash
rsync -avz --bwlimit=5000 src/ dest/
```

- `--bwlimit=5000` 限速 **≈5MB/s**
    

---

## 五、排除不需要的文件 / 目录

```bash
rsync -av --exclude='node_modules' --exclude='*.log' src/ dest/
```

常用排除：

- `--exclude='.git/'`
    
- `--exclude='target/'`
    
- `--exclude='*.tmp'`
    

---

## 六、只同步，不覆盖新文件

```bash
rsync -avu src/ dest/
```

- `-u` = update：只更新比目标旧的文件，不覆盖新文件
    

---

## 七、测试运行（不真拷贝）

```bash
rsync -avn src/ dest/
```

- `-n` = dry-run 只看会发生什么，非常安全
    

---

## 八、备份场景：保留旧版本（避免误删）

```bash
rsync -av --backup --backup-dir=backup/$(date +%Y%m%d) src/ dest/
```

会把被覆盖的文件自动移到按日期命名的备份目录

---

## 九、最常用万能组合（记这一条就够）

**本地同步**

```bash
rsync -av --delete src/ dest/
```

**远程传输 + 压缩 + 进度 + 断点续传**

```bash
rsync -avzP src/ user@ip:/path/
```