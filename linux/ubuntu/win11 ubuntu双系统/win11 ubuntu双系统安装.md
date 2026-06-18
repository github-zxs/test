#### **强烈建议先安装 Windows 11，再安装 Ubuntu。**
**为什么要先装 Windows 11？**
1. **引导机制决定**
    
    - Windows 的引导管理器（Windows Boot Manager）**无法识别 Linux 系统**，安装后会直接覆盖整个硬盘的引导记录，导致已安装的 Ubuntu 无法启动。
    - Ubuntu 使用 **GRUB 引导器**，安装时会自动检测并添加 Windows 启动项，形成统一的启动菜单，开机时可自由选择系统。
    
2. **分区结构影响**
    
    - Windows 11（UEFI 模式）安装时会自动创建 **EFI 系统分区、恢复分区**等关键分区。
    - 若先装 Ubuntu，再装 Windows，Windows 会覆盖 EFI 分区，导致 Ubuntu 引导丢失，修复难度大。
    
3. **操作更简单、风险更低**
    - 先装 Windows，再用 Ubuntu 安装程序识别 Windows，**自动配置双引导**，几乎不用手动干预。
    - 若先装 Ubuntu，后装 Windows，必须手动修复 GRUB，对新手不友好
#### win11 必须修改的两个设置
1. **关闭快速启动** 
    控制面板
    → 电源选项
    → 选择电源按钮功能
    → 更改当前不可用设置
    → 取消 **启用快速启动**
    
2. **关闭安全启动 Secure Boot**
    重启进 BIOS（一般 F2 / Del / F10 / F12）
    → 找到 Secure Boot → 设为 Disabled
3. 关闭安全启动（Secure Boot）
#### 删除ubuntu多引导菜单中无用的选先
```bash
sudo add-apt-repository ppa:danielrichter2007/grub-customizer
sudo apt update
sudo apt install grub-customizer -y
```

# Win11 + Ubuntu 双系统 **完整卸载 Ubuntu 教程**

全程不碰 `grub.cfg`、不搞危险操作，只删干净、不留坑。

---

## 一、卸载总思路（记住这 3 步）

1. **先修复 Windows 引导**（把开机控制权还给 Windows）
2. **再删除 Ubuntu 分区**
3. **最后清理 EFI 里的 Ubuntu 引导项**

---

# 二、正式步骤（照着做就行）

## 1. 进入 Windows 11

确保你现在在 **Windows 里**，不在 Ubuntu。

---

# 三、第一步：修复 Windows 主引导（最重要）

以 **管理员身份** 打开 **命令提示符（CMD）**

依次执行：

bash

运行

```
diskpart
```

然后输入：

bash

运行

```
list disk
```

找到你装系统的 **硬盘**（一般是 Disk 0）

bash

运行

```
select disk 0
list partition
```

找到 **EFI 分区**（大小一般 100MB 或 512MB，类型是 “系统”）

假设是 Partition 1，执行：

bash

运行

```
select partition 1
assign letter=O
exit
```

然后修复 Windows 引导：

bash

运行

```
bcdboot C:\Windows /s O: /f UEFI
```

✅ 出现 **成功创建启动文件** 就说明引导修复完成。

---

# 四、第二步：删除 Ubuntu 分区

1. 右键开始菜单 → **磁盘管理**
2. 找到这些分区（Linux 分区 Windows 不识别，会显示**基本数据分区**）：
    
    - / 根分区（ext4）
    - swap 分区
    - 如果你单独分了 /home 也要删
    
3. 右键 → **删除卷** → 是
4. 删除后会变成 **未分配空间**，你可以：
    
    - 扩展回 C 盘
    - 或新建 NTFS 数据盘
    

---

# 五、第三步：清理 EFI 里的 Ubuntu 引导项（防止开机乱跳）

重新以 **管理员** 打开 CMD：

bash

运行

```
diskpart
select disk 0
select partition 1
assign letter=O
exit
```

然后打开 O 盘：

bash

运行

```
O:
cd EFI
dir
```

你会看到 `ubuntu` 文件夹，直接删除：

bash

运行

```
 1.命令提示符（CMD）标题是「命令提示符」`rd /s /q O:\EFI\ubuntu`
 2.删除Ubuntu引导文件夹（PowerShell专用） Remove-Item -Path "O:\EFI\ubuntu" -Recurse -Force
```

完成后，卸载 O 盘（避免以后误删）：

bash

运行

```
diskpart
select disk 0
select partition 1
remove letter=O
exit
```

---

# 六、完成！

现在你的电脑：

- 开机 **直接进 Windows**
- 没有 GRUB 菜单
- Ubuntu 已**彻底删除**
- 空间完全回收
- 无任何引导风险