# Hyprland (Omarchy) Super+Space 菜单闪黑屏 2 秒排查与解决方案

---

## 1. 现象描述
* **触发操作**：按 `Super + Space` 快捷键调出系统主菜单（`omarchy-menu`）。
* **故障现象**：屏幕瞬间黑屏约 2 秒，伴随短暂卡顿/无信号状态，随后恢复正常并显示菜单。

---

## 2. 系统环境与硬件架构
* **操作系统 / 发行版**：Omarchy Linux (Arch Linux 系)
* **窗口管理器 / 合成器**：Hyprland (Wayland) + `aquamarine` 渲染后端
* **显卡与驱动**：NVIDIA GeForce RTX 2060 Mobile（`nvidia` 专有驱动，启用了 `nvidia_drm.modeset=1`）
* **显示设备**：外接 4K 显示器（Dell UP3218K @ 3840×2160@60Hz）
* **菜单前端组件**：基于 QuickShell 开发的 `omarchy-menu`

---

## 3. 技术根因分析

### (1) 全屏 Overlay 图层与 Direct Scanout 冲突
* `omarchy-menu` 采用 `WlrLayershell.layer: Overlay` 规范，唤出时会在屏幕最顶层覆盖一层全屏蒙版（`scrim`）和浮动卡片。
* 在 **NVIDIA 驱动 + Wayland** 组合下，当出现全屏半透明图层时，GPU 的直接扫描输出（Direct Scanout）会被打断，DRM/KMS 接口在图层重构时触发重新握手（KMS Mode Reset），导致显示器黑屏 1~2 秒。

### (2) 非整数缩放（Fractional Scaling）与图层动画的开销放大
* **非整数缩放**（如 1.25x、1.5x、1.75x）：在 NVIDIA 下合成非整数缩放的 Layer Surface 时，显卡需要实时进行双线性插值重采样，更容易引发显卡 Buffer 竞争与重置。
* **Layer 级淡入淡出动画**：`layersIn` / `fadeLayersIn` 动画过程中的连续 Alpha 渐变，使得合成器频繁请求新的 KMS Buffer，加剧了黑屏闪烁发生的概率。

---

## 4. 关键配置文件与修复方案

### ① 屏幕缩放配置：使用整数缩放
编辑 `~/.config/hypr/monitors.lua`，确保主显示器采用整数缩放倍率（如 2x）：

```lua
-- ~/.config/hypr/monitors.lua
local omarchy_gdk_scale = 2
local omarchy_monitor_scale = 2

hl.env("GDK_SCALE", tostring(omarchy_gdk_scale))
hl.monitor({ output = "DP-1", mode = "3840x2160@60", position = "0x0", scale = omarchy_monitor_scale })
hl.monitor({ output = "eDP-1", mode = "1920x1080@144", position = "0x0", scale = omarchy_monitor_scale, disabled = true })
```

### ② 视觉效果配置：禁用 Layer 动画
编辑 `~/.config/hypr/looknfeel.lua`，关闭图层级动画：

```lua
-- ~/.config/hypr/looknfeel.lua
-- Disable compositor layer fade animations.
-- 避免在 NVIDIA 显卡上唤出 Overlay 菜单时触发图层 Alpha 渐变导致的闪黑
hl.animation({ leaf = "layers",        enabled = false })
hl.animation({ leaf = "layersIn",      enabled = false })
hl.animation({ leaf = "layersOut",     enabled = false })
hl.animation({ leaf = "fadeLayersIn",  enabled = false })
hl.animation({ leaf = "fadeLayersOut", enabled = false })
```

---

## 5. 日常使用与维护建议
1. **多显示器 / 拔插屏幕**：若外接显示器或笔记本开合盖（Clamshell Mode），优先保持各显示器为整数倍率缩放（`scale = 1` 或 `scale = 2`）。
2. **动画调整**：若后续修改桌面主题或动画效果，请保留 `layers`、`fadeLayers` 等动画处于关闭状态，以保证在 NVIDIA 显卡下的极致响应与防闪黑体验。
