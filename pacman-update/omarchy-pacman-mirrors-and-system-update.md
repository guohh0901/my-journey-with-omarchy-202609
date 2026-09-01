# Omarchy 软件源换国内镜像加速与系统更新指南

---

## 1. 现象与问题

### 现象 ①：软件包下载速度极慢
在执行系统更新或安装软件时，下载速度仅有几十 KB/s 至 200 KB/s，下载较大更新（如 1GB+）耗时极长。

### 现象 ②：直接执行 `sudo pacman -Syu` 被拦截
尝试通过常规 Arch Linux 方式升级时，触发系统 Pre-transaction hook 拦截并报错：
```text
:: Running pre-transaction hooks...
(1/7) Checking Omarchy update entrypoint...

Woah partner...

This looks like a direct pacman system upgrade. Omarchy updates should normally
run through:

  omarchy update
...
error: command failed to execute correctly
error: failed to commit transaction (failed to run transaction hooks)
Errors occurred, no packages were upgraded.
```

---

## 2. 技术根因分析

### (1) 默认镜像源为国外单点服务器
Omarchy 默认在 `/etc/pacman.d/mirrorlist` 中仅配置了一个国外官方镜像源：
```ini
Server = https://stable-mirror.omarchy.org/$repo/os/$arch
```
由于基础仓库（`core`、`extra`、`multilib`）均从该国外服务器拉取，受网络链路限制导致吞吐量极低。

### (2) Omarchy 专属更新流与安全保护机制
Omarchy 在 Arch Linux 基础上增加了自动化运维体系。直接使用 `pacman -Syu` 会跳过 Omarchy 的生命周期管理。官方的 `omarchy update` 命令会自动执行：
* 系统自动快照（Snapshot）创建（便于出现问题时快速回滚）
* Omarchy 与 Arch Keyring 密钥链同步与校验
* 系统配置与版本环境迁移（Migrations）
* Mise 工具链与 AUR 软件包更新
* 升级后的服务重启（Restart Checks）与状态检查

因此，系统通过 `/usr/share/libalpm/hooks/` 设置了拦截钩子，强制规范更新入口。

---

## 3. 国内镜像源测速与配置

### 实测下载测速（测试样本：8.5 MB `extra.db` 仓库数据库）

| 镜像源 | 耗时 | 下载速率 | 状态 |
| :--- | :--- | :--- | :--- |
| **USTC (中科大)** | **0.26 s** | **32.05 MB/s** | 🚀 **实测最快（推荐置顶）** |
| **腾讯云 (Tencent Cloud)** | **0.50 s** | **16.88 MB/s** | 🚀 **极快** |
| **BFSU (北外镜像)** | **0.52 s** | **16.20 MB/s** | 🚀 **极快** |
| **清华大学 (TUNA)** | **1.16 s** | **7.28 MB/s** | 稳定 |
| **华为云 (Huawei Cloud)** | **1.31 s** | **6.46 MB/s** | 稳定 |
| **阿里云 (Aliyun)** | **9.88 s** | **0.86 MB/s** | 较慢 |
| **Omarchy 官方默认源** | **42.11 s** | **0.20 MB/s** | 🐢 极慢 |

### 镜像源配置方法

编辑或覆盖 `/etc/pacman.d/mirrorlist`：

```bash
sudo tee /etc/pacman.d/mirrorlist << 'EOF'
## USTC (中科大 - 实测最快)
Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch

## 腾讯云
Server = https://mirrors.cloud.tencent.com/archlinux/$repo/os/$arch

## 北外 BFSU
Server = https://mirrors.bfsu.edu.cn/archlinux/$repo/os/$arch

## 清华 TUNA
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch

## 华为云
Server = https://repo.huaweicloud.com/archlinux/$repo/os/$arch

## 阿里云
Server = https://mirrors.aliyun.com/archlinux/$repo/os/$arch

## 官方备用
Server = https://stable-mirror.omarchy.org/$repo/os/$arch
EOF
```

> **注意**：`/etc/pacman.conf` 中的 `[omarchy]` 专有源（`https://pkgs.omarchy.org/`）用于分发 Omarchy 自研组件，包体积小且国内访问延迟低（~0.5s），保留默认配置即可。

---

## 4. 标准更新流程与常用命令

### 步骤 ①：强制刷新本地数据库索引
更换镜像源后，强制刷新本地 pacman 数据库缓存：
```bash
sudo pacman -Syy
```

### 步骤 ②：执行 Omarchy 系统更新
使用 Omarchy 官方工具执行完整的升级流：

```bash
# 交互式升级（会提示确认）
omarchy update

# 或自动确认并升级
omarchy update -y
```

---

## 5. Omarchy 更新相关常用子命令

| 命令 | 说明 |
| :--- | :--- |
| `omarchy update` | 执行标准系统更新（包含快照、Keyring、pacman、Mise、迁移等） |
| `omarchy update available` | 检查是否有可用的系统及组件更新 |
| `omarchy update pkg prune` | 清理 pacman 缓存中已被替代的历史旧版本安装包 |
| `omarchy update orphan pkgs` | 检查并按需清理系统无用孤立包（Orphans） |
| `omarchy update restart` | 检查并在更新后提示重启受影响的系统服务或机器 |
| `omarchy update analyze logs`| 当更新发生异常时，分析更新日志排查故障 |

> **应急旁路（不建议日常使用）**：  
> 若在特殊调试场景下必须直接使用 `pacman` 且绕过 Omarchy 检查钩子，可使用：  
> `sudo env OMARCHY_ALLOW_DIRECT_PACMAN=1 pacman -Syu`
