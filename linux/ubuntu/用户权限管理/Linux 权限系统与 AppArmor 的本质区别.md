Linux AppArmor 与权限系统学习文档

# 一、核心概念：Linux 权限系统与 AppArmor 的本质区别

Linux 权限系统（DAC，自主访问控制）是 Linux 原生基础安全机制，AppArmor（MAC，强制访问控制）是基于内核的进程级安全增强工具，两者互补叠加，核心原则：**AppArmor 的权限不会超过 Linux 传统权限系统的上限**，无法突破 DAC 权限限制。

## 1.1 Linux 传统权限系统（DAC）

- **本质**：基于用户/组身份的资源访问控制，是所有 Linux 系统的基础安全机制。
    
- **控制对象**：用户/组对文件/目录的访问权限，核心是 `rwx`（读/写/执行）权限位。
    
- **决策逻辑**：谁拥有资源，谁决定权限（文件所有者可自由分配 `chmod`/`chown` 权限）。
    
- **核心模型**：所有者（u）+ 所属组（g）+ 其他用户（o）+ rwx 权限位。
    
- **作用**：实现用户间数据隔离、基础操作权限划分，满足日常文件管理需求。
    
- **特点**：粗粒度控制、默认宽松（未明确拒绝则允许）、用户主导。
    

## 1.2 AppArmor（MAC）

- **本质**：基于进程的强制访问控制，通过 Linux 安全模块（LSM）实现，属于内核级安全增强。
    
- **控制对象**：单个进程/程序能访问的所有系统资源（文件、网络、端口、IPC 等）。
    
- **决策逻辑**：系统通过策略文件（Profile）统一决定，与用户身份无关（同一程序，任何用户运行都受相同约束）。
    
- **核心模型**：为每个程序绑定专属 Profile 配置文件，遵循“默认拒绝”原则（未显式允许的操作均被拦截）。
    
- **作用**：遵循最小权限原则，限制程序行为，防御漏洞利用、恶意代码横向扩散，适用于高安全需求场景。
    
- **特点**：细粒度控制、默认严格、系统主导、叠加于 DAC 之上。
    

## 1.3 关键维度对比表

|对比维度|Linux 传统权限系统（DAC）|AppArmor（MAC）|
|---|---|---|
|控制粒度|粗粒度：用户/组 → 文件/目录|细粒度：进程 → 全系统资源|
|决策依据|用户ID、组ID + 文件权限位|进程路径 + AppArmor 策略规则|
|默认规则|未明确拒绝则允许（宽松）|未明确允许则拒绝（严格）|
|作用层级|用户空间 + 文件系统层|内核层（拦截系统调用）|
|权限继承|进程继承用户权限，易扩散|严格限制进程权限，阻断越权|
|配置方式|`chmod`/`chown`/`chgrp` 命令|编写 `/etc/apparmor.d/` 下的 Profile 文件|
|生效关系|基础权限，必须满足|叠加约束，权限上限不超过 DAC|
|典型场景|日常文件管理、用户隔离|服务器加固、容器安全、漏洞防护|

# 二、工作机制与生效流程

## 2.1 DAC 权限检查流程（基础）

1. 进程发起文件访问请求（如读、写、执行）。
    
2. 内核检查：进程的 UID/GID → 匹配文件的 u/g/o 权限组 → 验证对应的 rwx 权限。
    
3. 权限匹配则允许执行操作，否则返回 `Permission denied`（权限拒绝）。
    

## 2.2 AppArmor 检查流程（叠加 DAC）

1. 进程发起系统调用（文件访问、网络连接、IPC 等）。
    
2. 第一步：内核先执行 DAC 权限检查，若不通过则直接拒绝，AppArmor 不参与后续检查。
    
3. 第二步：DAC 检查通过后，内核查询 AppArmor，匹配该进程路径对应的 Profile 配置文件。
    
4. 第三步：检查当前操作是否在 Profile 的允许列表中：
    
    1. 允许：执行操作；
        
    2. 拒绝：拦截操作并记录日志（Enforcing 强制模式）。
        

## 2.3 实战示例：理解两者叠加关系

假设场景：

- 文件 `/var/log/nginx/access.log` 的 DAC 权限：`root:adm rw-r-----`（仅 root 和 adm 组可读写）。
    
- AppArmor 为 `nginx` 进程配置 Profile，仅允许读取该文件。
    

场景1：root 用户运行 nginx

- DAC 检查：root 拥有读写权限，检查通过；
    
- AppArmor 检查：仅允许读取 → 最终只能读，不能写（AppArmor 收紧权限）。
    

场景2：普通用户（非 adm 组）运行 nginx

- DAC 检查：普通用户无权限 → 直接拒绝，AppArmor 不生效。
    

结论：**AppArmor 是在 DAC 基础上的“收权”工具，无法突破 DAC 权限上限**。

# 三、AppArmor 核心操作命令（通用）

以下命令适用于所有 AppArmor 配置（Nginx、MySQL 等），Debian/Ubuntu 系统通用。

## 3.1 配置重载与生效

```bash
# 重新解析指定 Profile 配置（核心命令）
apparmor_parser -r /etc/apparmor.d/[进程名]
# 示例：重载 Nginx 配置
apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx
# 重载 AppArmor 服务（可选）
systemctl reload apparmor
```

## 3.2 模式切换（关键调试手段）

```bash
# 强制模式（Enforcing）：拦截违规操作并记录日志（生产环境常用）
aa-enforce [进程名]
# 示例：nginx 强制模式
aa-enforce usr.sbin.nginx

# 学习模式（Complain）：只记录违规操作，不拦截（调试、补全规则常用）
aa-complain [进程名]
# 示例：mysql 学习模式
aa-complain usr.sbin.mysqld
```

## 3.3 状态查看与日志排查

```bash
# 查看 AppArmor 整体状态（查看已加载的 Profile 及模式）
aa-status

# 查看 AppArmor 违规日志（两种方式）
dmesg | grep -i apparmor
journalctl -t audit -e  # 实时查看审计日志
journalctl -t audit -f  # 持续监控日志
```

## 3.4 自动补全规则（神器）

```bash
# 扫描系统日志，自动识别缺失的权限，交互式补全到 Profile 中
aa-logprof
```

使用场景：先将 Profile 设为学习模式（aa-complain），运行业务1-2天，再执行该命令，自动补全缺失权限，避免手动配置遗漏。

## 3.5 AppArmor 权限符号说明

- r：read（读取权限）
    
- w：write（写入权限）
    
- x：execute（执行权限）
    
- m：memory map（内存映射权限，常用于程序二进制文件）
    
- k：lock（文件锁权限，常用于数据库、日志文件）
    
- 组合使用：如 rw（读写）、mr（读取+内存映射）、rwk（读写+锁）
    

# 四、AppArmor 实战配置模板

以下模板均适用于 Debian/Ubuntu 系统，可直接复制使用，按需修改路径即可。

## 4.1 Nginx AppArmor 配置模板

### 配置文件路径

`/etc/apparmor.d/usr.sbin.nginx`（文件名对应 Nginx 进程路径 `/usr/sbin/nginx`）

### 完整配置内容

```aa
#include <tunables/global>

/usr/sbin/nginx {
  # 通用系统库与基础权限（按需启用，注释解开即可）
  #include <abstractions/base>
  #include <abstractions/nameservice>
  #include <abstractions/openssl>
  #include <abstractions/kerberosclient>

  # 允许读取 Nginx 自身二进制文件与配置文件
  /usr/sbin/nginx mr,
  /etc/nginx/** r,
  /usr/share/nginx/** r,

  # 网站根目录（核心，按需修改为你的实际网站路径）
  /var/www/html/** r,
  # /var/www/yourdomain/** r,  # 自定义网站目录示例

  # 日志文件读写权限
  /var/log/nginx/** rw,
  /var/log/nginx/*.log rw,

  # PID 文件与运行时目录
  /var/run/nginx.pid rw,
  /var/run/nginx/ rw,

  # 临时目录、缓存目录（Nginx 临时文件存放）
  /var/lib/nginx/** rw,
  /var/tmp/nginx/** rw,
  /tmp/nginx/** rw,
  /tmp/* rw,

  # 允许监听网络端口（http/https）
  network inet tcp,
  network inet6 tcp,
  network netlink raw,

  # 允许绑定 80（http）、443（https）端口
  tcp bind 80,
  tcp bind 443,

  # 允许子进程、信号通信等（Nginx 运行必需）
  signal receive peer=unconfined,
  signal send peer=unconfined,
  capability dac_override,
  capability dac_read_search,
  capability net_bind_service,
  capability setgid,
  capability setuid,
  capability chown,
  capability kill,

  # 允许动态模块加载（若使用 Nginx 动态模块需启用）
  /usr/lib/nginx/modules/*.so mr,

  # SSL 证书读取权限（按需修改路径，如 Let's Encrypt 证书）
  /etc/ssl/** r,
  /etc/letsencrypt/live/** r,
  /etc/letsencrypt/archive/** r,

  # 拒绝访问敏感路径（安全加固，禁止 Nginx 访问系统敏感文件）
  deny /etc/shadow rw,
  deny /etc/sudoers rw,
  deny /root/** rw,
}
```

## 4.2 MySQL AppArmor 配置模板（默认数据目录）

### 配置文件路径

`/etc/apparmor.d/usr.sbin.mysqld`（文件名对应 MySQL 进程路径 `/usr/sbin/mysqld`）

### 完整配置内容（默认数据目录：/var/lib/mysql）

```aa
#include <tunables/global>

/usr/sbin/mysqld {
  # 基础系统依赖（按需启用）
  #include <abstractions/base>
  #include <abstractions/nameservice>
  #include <abstractions/openssl>
  #include <abstractions/ssl_keys>
  #include <abstractions/user-tmp>

  # MySQL 自身二进制与配置文件
  /usr/sbin/mysqld mr,
  /etc/mysql/** r,
  /usr/share/mysql/** r,
  /usr/lib/mysql/** mr,
  /etc/mysql/mysql.conf.d/** r,

  # 数据目录（核心，默认路径）
  /var/lib/mysql/ r,
  /var/lib/mysql/** rwk,

  # 临时文件 & 缓存目录
  /tmp/ r,
  /tmp/** rwk,
  /var/tmp/ r,
  /var/tmp/** rwk,

  # 日志文件
  /var/log/mysql/ r,
  /var/log/mysql/** rw,

  # PID 文件 & 运行时目录
  /run/mysqld/ rw,
  /run/mysqld/mysqld.pid rw,
  /run/mysqld/mysqld.sock rw,
  /run/mysqld/mysqld.sock.lock rwk,

  # MySQL 运行必需的系统能力
  capability dac_override,
  capability dac_read_search,
  capability setgid,
  capability setuid,
  capability net_bind_service,
  capability chown,
  capability fowner,
  capability fsetid,

  # 网络权限（允许 MySQL 监听和连接）
  network inet tcp,
  network inet6 tcp,
  network unix stream,

  # SSL 证书读取权限
  /etc/ssl/** r,
  /var/lib/mysql-files/** r,

  # 拒绝敏感路径（安全加固）
  deny /etc/shadow rw,
  deny /etc/sudoers rw,
  deny /root/** rwkl,
}
```

## 4.3 MySQL AppArmor 配置模板（自定义数据目录）

### 适用场景

当 MySQL 数据目录不在默认的 `/var/lib/mysql` 时，修改以下模板中的数据目录路径即可直接使用。

### 完整配置内容（自定义数据目录）

```aa
#include <tunables/global>

/usr/sbin/mysqld {
  # 基础系统依赖（按需启用）
  #include <abstractions/base>
  #include <abstractions/nameservice>
  #include <abstractions/openssl>
  #include <abstractions/ssl_keys>
  #include <abstractions/user-tmp>

  # MySQL 自身二进制与配置文件
  /usr/sbin/mysqld mr,
  /etc/mysql/ r,
  /etc/mysql/** r,
  /usr/share/mysql/** r,
  /usr/lib/mysql/** mr,
  /usr/lib/x86_64-linux-gnu/libmysql* mr,

  # ==============================================
  # 自定义 MySQL 数据目录（替换为你的真实路径）
  # ==============================================
  /your/custom/mysql/data/ r,
  /your/custom/mysql/data/** rwk,

  # 临时文件 & 缓存目录
  /tmp/ r,
  /tmp/* rwk,
  /var/tmp/ r,
  /var/tmp/* rwk,

  # 日志文件
  /var/log/mysql/ r,
  /var/log/mysql/** rw,

  # PID 文件 & 运行时目录（若路径不同，同步修改）
  /run/mysqld/ rw,
  /run/mysqld/mysqld.pid rw,
  /run/mysqld/mysqld.sock rw,
  /run/mysqld/mysqld.sock.lock rwk,

  # 网络权限
  network inet tcp,
  network inet6 tcp,
  network unix stream,

  # MySQL 运行必需的系统能力
  capability dac_override,
  capability dac_read_search,
  capability setgid,
  capability setuid,
  capability net_bind_service,
  capability chown,
  capability fowner,
  capability fsetid,

  # SSL 证书读取权限
  /etc/ssl/** r,

  # 拒绝敏感路径（安全加固）
  deny /etc/shadow r,
  deny /etc/sudoers r,
  deny /root/** rwkl,
}
```

### 修改说明

- 核心修改：将 `/your/custom/mysql/data` 替换为你的 MySQL 真实数据目录（如 `/data/mysql`）。
    
- 可选修改：若 MySQL 的 PID 文件、sock 文件不在 `/run/mysqld`，同步修改对应路径。
    

# 五、实战注意事项

1. **权限叠加原则**：始终记住 AppArmor 无法突破 DAC 权限，若 DAC 权限不足，即使 AppArmor 允许，操作也会被拒绝。
    
2. **配置调试流程**：生产环境配置时，建议先设为学习模式（aa-complain），运行1-2天业务，用 `aa-logprof` 补全权限，再切换为强制模式（aa-enforce），避免遗漏权限导致程序无法运行。
    
3. **路径匹配规则**：Profile 中路径末尾的 `/` 和 `**` 含义不同：`/path/` 表示目录本身，`/path/**` 表示目录下所有文件和子目录。
    
4. **敏感路径加固**：务必保留“deny”规则，禁止程序访问 `/etc/shadow`、`/etc/sudoers` 等系统敏感文件，提升安全性。
    
5. **MariaDB 适配**：MariaDB 与 MySQL 进程路径一致（`/usr/sbin/mysqld`），上述 MySQL 模板可直接用于 MariaDB，只需确认数据目录、配置目录路径正确即可。
    

# 六、总结

1. Linux 传统权限系统（DAC）：解决“谁能访问文件”的问题，是基础、宽松、用户主导的权限控制，适用于日常文件管理和用户隔离。

2. AppArmor（MAC）：解决“这个进程能做什么”的问题，是增强、严格、系统主导的进程约束，适用于服务器加固、容器安全、漏洞防护等场景。

3. 两者关系：互补叠加，AppArmor 不替代 DAC，而是在 DAC 基础上对进程权限进行更细粒度的收紧，提升系统安全等级。

4. 实战核心：掌握 Profile 配置、模式切换、日志排查和权限补全命令，按需修改路径，即可快速实现 Nginx、MySQL 等程序的 AppArmor 安全加固。