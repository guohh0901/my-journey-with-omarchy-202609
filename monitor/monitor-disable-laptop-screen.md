# 关闭笔记本内置屏，只保留 DP-1 外接显示器

> 日期：2026
> 系统：Omarchy (Hyprland 0.56.2)
> 目标：两块屏幕中只使用 DP-1（Dell UP3218K 外接 4K），关闭 eDP-1（笔记本内置屏）

---

## 1. 初始环境

查看当前所有显示器：

```bash
hyprctl monitors all
```

输出关键信息：

| 显示器 | 名称 | 分辨率 | 位置 | 缩放 | 状态 |
|--------|------|--------|------|------|------|
| 外接 Dell UP3218K | DP-1 | 3840x2160 @ 60Hz | 0x0 | 1.25 | 启用，聚焦 |
| 笔记本内置屏 | eDP-1 | 1920x1080 @ 144Hz | 3072x0 | 1.25 | 启用 |

原始配置文件 `~/.config/hypr/monitors.lua`：

```lua
hl.env("GDK_SCALE", tostring(omarchy_gdk_scale))
hl.monitor({ output = "", mode = "preferred", position = "auto", scale = omarchy_monitor_scale })
```

> 空字符串 `output = ""` 表示 Hyprland 自动应用配置到所有连接的显示器。

---

## 2. 备份原配置

```bash
cp ~/.config/hypr/monitors.lua ~/.config/hypr/monitors.lua.bak.$(date +%s)
```

实际生成：`monitors.lua.bak.1787728289`

---

## 3. 修改配置

### 3.1 第一次尝试（失败 ❌）

```lua
hl.monitor({ output = "DP-1", mode = "3840x2160@60", position = "0x0", scale = omarchy_monitor_scale })
hl.monitor({ output = "eDP-1", mode = "1920x1080@144", position = "0x0", scale = omarchy_monitor_scale, disable = true })
```

执行 `hyprctl reload && hyprctl configerrors` 报错：

```
/home/guohh/.config/hypr/monitors.lua:9: hl.monitor: unknown field 'disable'
```

**原因**：Hyprland 原生 monitor 配置字段是 `disabled`，不是 `disable`。

### 3.2 第二次尝试（成功 ✅）

把 `disable` 改成 `disabled`：

```lua
hl.monitor({ output = "DP-1", mode = "3840x2160@60", position = "0x0", scale = omarchy_monitor_scale })
hl.monitor({ output = "eDP-1", mode = "1920x1080@144", position = "0x0", scale = omarchy_monitor_scale, disabled = true })
```

重载并验证：

```bash
hyprctl reload && hyprctl configerrors
```

输出：

```
ok
```

无报错。

---

## 4. 验证结果

执行 `hyprctl monitors all`，eDP-1 状态变为：

```
Monitor eDP-1 (ID 1):
        1920x1080@144.10400 at 0x0
        ...
        active workspace: 0 ()           # 之前是 2
        ...
        disabled: true                   # ✅ 已禁用
        ...
```

DP-1 保持正常启用并聚焦。

---

## 5. 最终配置

`~/.config/hypr/monitors.lua`：

```lua
-- See https://wiki.hypr.land/Configuring/Basics/Monitors/
-- List current monitors and supported resolutions with: hyprctl monitors all

local omarchy_gdk_scale = 1
local omarchy_monitor_scale = 1.25

hl.env("GDK_SCALE", tostring(omarchy_gdk_scale))
hl.monitor({ output = "DP-1", mode = "3840x2160@60", position = "0x0", scale = omarchy_monitor_scale })
hl.monitor({ output = "eDP-1", mode = "1920x1080@144", position = "0x0", scale = omarchy_monitor_scale, disabled = true })

-- Configure a specific monitor.
-- hl.monitor({ output = "DP-2", mode = "2560x1440@144", position = "0x0", scale = 1 })

-- Portrait/rotated secondary monitor (transform: 1 = 90°, 3 = 270°).
-- hl.monitor({ output = "DP-2", mode = "preferred", position = "auto", scale = 1, transform = 1 })
```

---

## 6. 恢复方法

要重新启用笔记本屏，把 `disabled = true` 改成 `false` 或删除该行，然后：

```bash
hyprctl reload && hyprctl configerrors
```

如果配置改坏了想恢复原状：

```bash
cp ~/.config/hypr/monitors.lua.bak.1787728289 ~/.config/hypr/monitors.lua
hyprctl reload
```

或者用 omarchy 自带命令重置整个 hyprland 配置（**会丢失所有自定义**）：

```bash
omarchy refresh hyprland
```

---

## 7. 关键经验

1. **Hyprland 禁用显示器的字段是 `disabled`，不是 `disable`** —— 字段名拼写错会导致 `hl.monitor: unknown field` 错误。
2. **Hyprland 会默认启用所有连接的显示器** —— 不写配置不等于禁用，必须显式 `disabled = true`。
3. **改完配置必须双重验证**：`hyprctl reload` 重载 + `hyprctl configerrors` 检查错误。
4. **改前备份是好习惯**：`cp file file.bak.$(date +%s)` 时间戳备份便于回溯。

---

## 8. 相关命令速查

```bash
# 查看所有显示器及状态
hyprctl monitors all

# 仅查看某个显示器（如 eDP-1）
hyprctl monitors all | sed -n '/^Monitor eDP-1/,/^$/p'

# 临时关闭某个显示器（不写配置）
hyprctl dispatch exec hyprctl keyword monitor "eDP-1, disable"

# 重载配置 + 验证
hyprctl reload && hyprctl configerrors

# 显示器聚焦切换
hyprctl dispatch focusmonitor eDP-1
hyprctl dispatch focusmonitor DP-1
```
