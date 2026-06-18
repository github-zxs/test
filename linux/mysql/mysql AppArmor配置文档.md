我直接**把你整套配置里每一行都用大白话翻译一遍**，包括你问的 PID 文件、`**`、`k` 权限，一次性彻底讲透，保证你以后看 AppArmor 跟看说明书一样。

# 一、先把关键符号彻底讲死（背会这一段就够）

## 1. 路径符号

- `/dir/`→ 只代表**目录本身**，不包含里面内容→ 程序要进这个目录，必须先给这行权限
    
- `/dir/*`→ 目录下**直接一层**文件 / 子目录，不递归
    
- `/dir/**`→ **目录下所有内容，递归到底**→ 所有文件、所有子目录、子子子目录全都包含→ MySQL 数据目录必须用这个
    

## 2. 权限符号

- `r` = read 读
- `w` = write 写
- `k` = lock 加文件锁（多进程安全用）
- `m` = memory map 内存映射（加载二进制、库文件用）
- `x` = execute 执行

---

# 二、你的 MySQL 配置逐行大白话翻译

aa

```aa
/data/mysql/data/ r,
```

→ 允许 mysqld 进程**进入 /data/mysql/data/ 这个目录**→ 只给 “进门权限”，不给碰里面东西

aa

```aa
/data/mysql/data/** rwk,
```

→ 允许对 `/data/mysql/data/` 下面**所有文件、所有子目录**递归操作：

- `r` 读数据文件
- `w` 写入、修改数据
- `k` 加文件锁（MySQL 多线程必须，否则会锁冲突、启动失败）

---

# 三、你的 Nginx PID 配置逐行大白话

aa

```aa
/var/run/nginx.pid rwk,
```

## 逐字拆：

- `r`：读取当前 Nginx 主进程 PID 号
- `w`：写入 PID、更新 PID、删除 PID
- `k`：**给文件加锁**

## 为什么 PID 文件要 `k`？

Nginx 是**典型多进程架构**：

- 1 个 master 主进程
- 多个 worker 子进程

它们都会访问 / 修改 `nginx.pid`。如果不加锁 `k`：

- 多个进程同时写同一个文件
- 内容会错乱、覆盖
- 导致：启动失败、重复启动、重启异常

**所以 PID 文件、socket 文件、lock 文件都必须加 k**

---

# 四、我把你整套 Nginx + MySQL 关键行全部翻译一遍

## Nginx 关键配置白话版

aa

```aa
/usr/sbin/nginx mr,
```

→ 读 + 内存映射→ 让系统把 Nginx 二进制加载进内存运行

aa

```aa
/etc/nginx/** r,
```

→ 读取所有配置文件，只读不写

aa

```aa
/var/log/nginx/** rw,
```

→ 日志读写，多进程场景建议改成 `rwk` 更稳

aa

```aa
/var/run/nginx.pid rwk,
```

→ PID 文件：读、写、加锁（多进程必备）

aa

```aa
/tmp/nginx/** rw,
```

→ 临时文件读写

---

## MySQL 关键配置白话版

aa

```aa
/usr/sbin/mysqld mr,
```

→ 读 + 内存映射→ 把 MySQL 加载进内存运行

aa

```aa
/etc/mysql/** r,
```

→ 只读配置文件

aa

```aa
/data/mysql/data/ r,
```

→ 允许进数据目录

aa

```aa
/data/mysql/data/** rwk,
```

→ 所有数据文件：读、写、加锁（数据库核心权限）

aa

```aa
/run/mysqld/mysqld.sock rw,
```

→ 本地连接用的 socket 文件

aa

```aa
/run/mysqld/mysqld.sock.lock rwk,
```

→ socket 锁文件：必须加 k，否则多进程冲突

aa

```aa
/tmp/** rwk,
```

→ 临时文件读写 + 锁

---

# 五、超级浓缩记忆口诀（你记这个）

1. **二进制文件、so 库 → 必须 `mr`**
2. **数据目录、socket、lock、PID → 必须 `k`**
3. **目录进门权限 → 必须 `/dir/ r,`**
4. **目录下所有内容递归 → 用 `/**`**
5. **数据库、Web 服务器多进程 → 凡是会被抢着写的文件都加 `k`**

# 六 停用 apparmor
```bash
# 停止 AppArmor 服务
sudo systemctl stop apparmor

# 卸载所有 AppArmor 配置（可选，彻底释放限制）
sudo aa-teardown

# 验证状态（显示 inactive 即已关闭）
sudo systemctl status apparmor
```

# 七 AppArmor 共有**5 种核心运行模式**，结合你遇到的状态，给你清晰说明

## 1. **Enforce 强制模式**

- 标识：`enforce mode`
- 严格拦截未授权操作，同时记录日志
- 生产环境正式防护使用

## 2. **Complain 宽容模式**

- 标识：`complain mode`
- **仅记录违规行为，不拦截**，用于调试配置规则
- 对应你之前看到的状态提示

## 3. **Unconfined 未受限模式**

- 存在配置文件，但进程未应用策略
- 常见于加载策略前已启动进程，完全不受管控

## 4. **Disable 禁用模式**

- 配置文件被禁用，不会对进程做任何限制
- 配置不会被 AppArmor 加载

## 5. **Kill 杀死模式**

- 检测到违规操作时**直接终止进程**，安全等级最高
- 适用于极高安全要求的场景，极少使用

---

### 常用模式切换命令

bash

运行

```bash
# 切换为强制模式
sudo aa-enforce <程序路径>

# 切换为宽容模式
sudo aa-complain <程序路径>

# 禁用配置
sudo aa-disable <配置文件路径>
```

### 快速查看所有模式状态

bash

运行

```bash
aa-status
```