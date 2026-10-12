# Omarchy 笔记本风扇狂转与 CPU 持续高热排查与解决方案（mise 包装脚本死循环与电源策略）

---

## 1. 现象描述

* **故障现象**：在日常使用中（未运行高负载编译、大型游戏或深度计算任务），笔记本散热风扇持续高转速狂转、噪音剧烈，笔记本机身温度显著升高。
* **硬件监控指标**：
  * **CPU 核心温度**：Package 温度高达 **89°C ~ 95°C**（接近 100°C 临界警戒线），各核心普遍在 80°C ~ 94°C 高温区间。
  * **CPU 运行频率**：全核心频率被锁定在 **3.7 GHz ~ 4.4 GHz** 持续高频，无法回落到低功耗空闲频率。
  * **显卡状态**：独立显卡 NVIDIA GeForce RTX 2060 Mobile 处于 P0 性能状态，功耗约 26W，温度达 **76°C**。
  * **系统负载**：`uptime` 显示系统平均负载（Load Average）长期维持在 **3.5 ~ 3.8** 左右。由于笔记本内部 CPU 与 GPU 通常共享热管散热系统，高热叠加导致整机温控模块持续驱动风扇全速运转。

---

## 2. 系统环境与硬件架构

* **操作系统 / 发行版**：Omarchy Linux (基于 Arch Linux 体系)
* **桌面环境 / 合成器**：Hyprland (Wayland)
* **中央处理器 (CPU)**：Intel Core 系列（8 核 16 线程，`intel_pstate` 调频驱动）
* **图形处理器 (GPU)**：NVIDIA GeForce RTX 2060 Mobile（`nvidia` 官方闭源驱动）
* **多版本运行环境管理**：`mise` (运行时工具包管理器)

---

## 3. 技术根因深度排查

### (1) 核心根因：`~/.local/bin/` 下 mise 包装脚本陷入无限自身递归调用（Fork-Exec Loop）

通过 `ps aux --sort=-%cpu` 与 `pstree` 排查，定位到了系统负载居高不下的最直接元凶：

| PID | CPU 占用率 | 累计占用 CPU 时间 | 进程启动命令 |
| :--- | :--- | :--- | :--- |
| **`231237`** | ~43.5% | 55+ 分钟 | `/bin/bash /home/guohh/.local/bin/gh api --hostname github.com rate_limit` |
| **`211632`** | ~42.3% | 53+ 分钟 | `/bin/bash /home/guohh/.local/bin/codex --version` |

这两个进程累计独占了将近 2 个 CPU 核心 100% 的满载算力，并且已经连续死循环空转近 1 小时。

#### 为什么简单的 `--version` 与 `rate_limit` 会吃满 CPU？
这两个进程由 Orca IDE 在后台启动时进行工具链环境探测所触发。查看原始包装脚本（如 `~/.local/bin/gh` 和 `~/.local/bin/codex`）：

```bash
#!/bin/bash
export MISE_MINIMUM_RELEASE_AGE=0
mise use -g "gh" || exit 1
exec mise x "gh" -- "gh" "$@"
```

* **死循环机制剖析**：
  1. 在子进程或 IDE 传递的环境变量中，`PATH` 头部包含 `/home/guohh/.local/bin`。
  2. 脚本在最后一步调用 `exec mise x "gh" -- "gh"` 时，`mise x` 在当前 `PATH` 中寻址可执行文件 `gh`。
  3. 由于 `~/.local/bin` 在 `PATH` 中优先级在最前，`mise` 再次命中了 `/home/guohh/.local/bin/gh` 脚本自身！
  4. 脚本再次被 `exec` 执行，再次调用 `exec mise x "gh" -- "gh"`，形成了 **Fork-Exec 级无限自身递归调用**。
  5. 监控 `/proc/231237/stat` 发现每秒发生近 **80,000 次**缺页异常与进程替换，瞬间把单核 CPU 彻底占满。
  6. 同样的问题潜在存在于 `~/.local/bin/` 下所有使用相同写法的包装工具（如 `claude`, `codex`, `copilot`, `gemini`, `opencode`, `pi` 等）。

### (2) 次要原因：系统电源管理策略锁定为 `performance`

通过 `powerprofilesctl get` 排查发现：
* 当前系统的全局电源策略处于 `performance` 模式。
* 在该模式下，Intel CPU 的 `energy_performance_preference`（EPP）被强制设定为 `performance`，各核心无法自动降频至 800MHz 左右的深层省电状态，全核持续运行在 4.2 GHz 以上。
* 加之独显处于 P0 性能状态（26W 恒定功耗），基础发热量进一步堆高了整体待机温度。

---

## 4. 修复方案与实施步骤

### 步骤一：紧急终止卡死进程
立即杀死陷入死循环的两个进程，释放 CPU 占用：

```bash
kill -9 231237 211632
```

### 步骤二：重构 `~/.local/bin/` 下的全部包装脚本
对 `~/.local/bin/` 涉及的 16 个工具（`gh`, `codex`, `claude`, `copilot`, `crush`, `gemini`, `opencode`, `pi`, `playwright`, `omp`, `grok`, `ghui`, `hunk`, `hermes`, `cursor-agent`, `muse`）进行改造。

1. **备份原始脚本**：
   ```bash
   mkdir -p ~/.local/bin/backup_wrappers_20261012
   cp -a ~/.local/bin/{gh,codex,claude,copilot,crush,gemini,opencode,pi,playwright,omp,grok,ghui,hunk,hermes,cursor-agent,muse} ~/.local/bin/backup_wrappers_20261012/
   ```

2. **采用“直连二进制 + 环境变量安全剥离”双重防护架构**：
   * **优先直连底层二进制（Zero Overhead）**：利用 `mise which <cmd>` 直接获取底层实际二进制的绝对路径并 `exec "$_bin" "$@"`，跳过 PATH 查找，仅耗时十余毫秒。
   * **PATH 剥离兜底（Prevent Recursion）**：如果走 fallback，必须先在子进程环境中剥离 `~/.local/bin`，确保任何时候都不会在 PATH 中循环找到脚本自己。

脚本通用模板如下：

```bash
#!/bin/bash
export MISE_MINIMUM_RELEASE_AGE=0

# 1. 尝试直接执行底层真实二进制文件，避免经过 PATH 查找
_bin="$(mise which "<cmd>" 2>/dev/null)"
if [[ -n "$_bin" && -x "$_bin" && "$_bin" != "$0" ]]; then
  exec "$_bin" "$@"
fi

# 2. 剥离 PATH 中的 ~/.local/bin，彻底杜绝递归自调用死循环
_clean_path=""
_old_ifs="$IFS"
IFS=":"
for _p in $PATH; do
  [[ "$_p" != "$HOME/.local/bin" && "$_p" != "/home/guohh/.local/bin" && -n "$_p" ]] && _clean_path="${_clean_path:+$_clean_path:}$_p"
done
IFS="$_old_ifs"
export PATH="$_clean_path"

mise use -g "<pkg>" || exit 1
exec mise x "<pkg>" -- "<cmd>" "$@"
```

### 步骤三：调整电源管理配置为 `balanced`
将系统电源模式切换为兼顾性能与功耗的平衡模式：

```bash
powerprofilesctl set balanced
```
切换后，CPU 调频调节器策略切换至 `balance_performance`，在空闲和轻负载时允许 CPU 动态降频至 800MHz~1.5GHz，大幅降低待机温度与电压。

---

## 5. 修复效果与验证对比

### (1) 硬件温度与功耗降幅

| 监控指标 | 修复前（异常状态） | 修复后（正常状态） | 改善幅度 |
| :--- | :--- | :--- | :--- |
| **CPU Package 温度** | **+95.0°C** | **+69.0°C**（核心 ~66°C） | 净降 **26°C** |
| **NVMe 固态硬盘温度** | **+69.8°C** | **+54.9°C** | 净降 **15°C** |
| **NVIDIA 独立显卡温度** | **+76.0°C** | **+66.0°C** | 净降 **10°C** |
| **系统负载 Load Average** | **3.84** | **1.37** | 恢复正常低水位 |
| **CPU 空闲比例** | 8.8% | **91.2%** | 彻底释放计算核心 |

### (2) 包装脚本执行耗时复测
* `codex --version`：**16 ms** 执行完毕并安全退出。
* `claude --version`：**18 ms** 执行完毕并安全退出。
* `gh api rate_limit`：**0.9 s** 正常获取 API 配额并安全退出。

---

## 6. 总结与日常避坑建议

1. **避免在包装脚本中盲目使用与脚本同名的命令**：在编写位于 `~/.local/bin` 的 Wrapper 脚本时，如果使用 `exec <cmd>`，务必确保调用的不是当前脚本本身。优先使用绝对路径或在调用前去除当前目录。
2. **警惕 IDE / 编辑器的后台常驻探测任务**：诸如版本号探测、API 速率检测等轻量任务，一旦遇到阻塞或递归脚本，会在后台无感消耗大量算力，导致整机发热甚至续航断崖式下跌。
3. **结合电源管理与温度监控定位发热**：当发现笔记本风扇异常长鸣时，善用 `sensors`、`uptime` 与 `ps aux --sort=-%cpu` 可以第一时间锁定发热根源。
