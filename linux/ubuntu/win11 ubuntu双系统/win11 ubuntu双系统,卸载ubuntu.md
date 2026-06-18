我帮你把**彻底卸载 Ubuntu 的核心安全规则 + 最终可执行步骤** 整理成「零失误清单」，你照着做，既不会碰危险文件，也不会删错东西：

# 彻底卸载 Ubuntu 安全操作清单（Win11+SSD 双系统）

## 一、绝对禁止的 3 件事（记死，碰了必翻车）

1. ❌ 绝对不手动编辑 `/boot/grub/grub.cfg`（自动生成，改了白改还会引导崩溃）；
2. ❌ 不删 Windows 引导项（`Windows Boot Manager`），只删 Ubuntu 相关；
3. ❌ 不先删 Ubuntu 分区再修引导（会直接卡 `grub>` 黑屏）。

## 二、正确卸载步骤（仅 4 步，全程在 Windows 操作）

### 步骤 1：以管理员身份打开「命令提示符（CMD）」（别用 PowerShell）

### 步骤 2：修复 Windows 引导 + 分配 EFI 盘符

cmd

```
diskpart
select disk 0
list partition
select partition 1  # 选标"系统"、大小100/512MB的EFI分区
assign letter=O
exit
bcdboot C:\Windows /s O: /f UEFI  # 修复引导，出现"成功创建启动文件"即可
```

### 步骤 3：删除 EFI 里的 Ubuntu 引导（只删这一个，别的不动）

cmd

```
rd /s /q O:\EFI\ubuntu  # CMD专用命令，无报错. (pwoershell用  Remove-Item -Path "O:\EFI\ubuntu" -Recurse -Force)
```

### 步骤 4：清理收尾

1. 卸载 EFI 盘符（避免误操作）：
    
    cmd
    
    ```
    diskpart
    select disk 0
    select partition 1
    remove letter=O
    exit
    ```
    
2. 删除 Ubuntu 分区：右键开始菜单 → 磁盘管理 → 找到 Linux 分区（无盘符、Windows 不识别）→ 右键删除卷 → 空间可扩展到 C 盘或新建分区。

## 总结

1. 卸载核心是「先清 Ubuntu 引导，再删分区」，不碰 `grub.cfg`；
2. 仅删除 EFI 分区下的 `ubuntu` 文件夹，绝不碰 `Microsoft` 文件夹；
3. 全程用 CMD 执行命令，避免 PowerShell 参数报错。

执行完重启电脑，会直接进入 Windows，GRUB 菜单、Ubuntu 残留全部消失，无任何引导风险。