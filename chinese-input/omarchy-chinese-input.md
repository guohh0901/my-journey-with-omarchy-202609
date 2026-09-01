# Omarchy 安装中文输入法（RIME 明月双拼 + Emoji）

> 在 Omarchy（Arch + Hyprland）上从零安装 fcitx5 + RIME，并配置微软双拼方案、简体输出、Emoji 补全、ori 皮肤、Hyprland 自启。

---

## 1. 安装包

```bash
# 官方仓库
sudo pacman -S --needed fcitx5 fcitx5-rime fcitx5-configtool fcitx5-gtk fcitx5-qt \
                   noto-fonts-cjk wqy-zenhei wqy-microhei \
                   rime-double-pinyin rime-emoji

# AUR 皮肤（omarchy 内置包装）
omarchy pkg aur add fcitx5-skin-ori-git
```

包说明：

| 包 | 作用 |
| --- | --- |
| `fcitx5` | 主框架 |
| `fcitx5-rime` | RIME 中州韵引擎 |
| `fcitx5-configtool` | 图形化配置（kcm_fcitx5） |
| `fcitx5-gtk` / `fcitx5-qt` | GTK/Qt 输入支持 |
| `noto-fonts-cjk` / `wqy-zenhei` / `wqy-microhei` | 中文字体（避免半字宽问题） |
| `rime-double-pinyin` | 双拼方案（含 double_pinyin_mspy / flypy / pyjj / abc 等） |
| `rime-emoji` | Emoji 补全数据 + opencc 配置 |
| `fcitx5-skin-ori-git` | ori 皮肤（AUR） |

---

## 2. 启用 zh_CN locale

```bash
sudo sed -i 's/^#zh_CN.UTF-8 UTF-8/zh_CN.UTF-8 UTF-8/' /etc/locale.gen
sudo locale-gen
```

确认：

```bash
locale -a | grep zh_CN
# zh_CN.utf8
```

---

## 3. 写入 RIME 用户配置

配置目录：`~/.local/share/fcitx5/rime/`

### 3.1 默认方案列表 `default.custom.yaml`

```yaml
patch:
  schema_list:
    - schema: luna_pinyin_simp     # 全拼简体（默认）
    - schema: double_pinyin_mspy   # 微软双拼（备选）
```

> 通过 `Ctrl+\`` 调起方案菜单可在两者之间切换。

### 3.2 给双拼加 Emoji 补全 `double_pinyin_mspy.custom.yaml`

```yaml
patch:
  # 引入 emoji 补全片段（来自 rime-emoji 包）
  __include: emoji_suggestion:/patch
```

> rime-emoji 包在 `/usr/share/rime-data/emoji_suggestion.yaml` 提供 patch 片段，在 `/usr/share/rime-data/opencc/emoji.json` 提供 opencc 配置，通过 `__include` 引入即可。

### 3.3 Emoji 开关说明（可选 `emoji_suggestion.custom.yaml`）

rime-emoji 默认快捷键为 `Ctrl+Alt+E`（在候选栏显示 🈶️），按一次关闭，再按一次开启。也可在 schema 里直接默认开启：

```yaml
patch:
  switches/@next:
    name: emoji_suggestion
    reset: 0              # 0 = 默认开启
    states: [ "🈚︎", "🈶️" ]
```

---

## 4. 配置 Fcitx5

### 4.1 添加 RIME 到输入法 `~/.config/fcitx5/profile`

```ini
[Groups/0]
# Group Name
Name=Default
# Layout
Default Layout=us
# Default Input Method
DefaultIM=rime

[Groups/0/Items/0]
# Name
Name=keyboard-us
# Layout
Layout=

[Groups/0/Items/1]
# Name
Name=rime
# Layout
Layout=

[GroupOrder]
0=Default
```

> 也可以运行 `fcitx5-configtool`，左侧找到 `Rime` 点 `>` 加到右边，**取消勾选** "Share input state among all programs"（Wayland 下该选项有 bug）。

### 4.2 设置皮肤 `~/.config/fcitx5/conf/classicui.conf`

```ini
[ClassicUI]
Theme=ori
```

---

## 5. Hyprland 自启

`~/.config/hypr/autostart.lua`：

```lua
-- Extra autostart processes.
o.launch_on_start("fcitx5 -d")
```

Wayland 下 fcitx5 一般能自动发现，**无需**手动设 `GTK_IM_MODULE` 等环境变量。少数应用（JetBrains、部分老 Qt 应用）调不出来时再在 `~/.config/hypr/hyprland.conf` 加：

```ini
env = GTK_IM_MODULE=fcitx5
env = QT_IM_MODULE=fcitx5
env = SDL_IM_MODULE=fcitx5
env = XMODIFIERS=@im=fcitx5
```

---

## 6. 立即启动 / 验证

```bash
# 启动
fcitx5 -d

# 验证 RIME 已加载
fcitx5-remote -n
# rime

# 验证当前方案
gdbus call --session --dest org.fcitx.Fcitx5 --object-path /rime \
  --method org.fcitx.Fcitx.Rime1.GetCurrentSchema
# ('double_pinyin_mspy',)

# 切换方案
fcitx5-remote -s double_pinyin_mspy
fcitx5-remote -s luna_pinyin_simp
```

---

## 7. 使用速查

| 操作 | 按键 |
| --- | --- |
| 中/英切换 | `Ctrl+Space` |
| RIME 方案菜单 | `F4` 或 `Ctrl+\`` |
| 简繁切换 | `Ctrl+Shift+F` |
| Emoji 补全开关 | `Ctrl+Alt+E` |
| 二三候选 | `=` / `Shift+Space` |
| 重新部署 | `F5`（或删 `~/.local/share/fcitx5/rime/build` 后重启） |

---

## 8. 故障排查

```bash
# 看 fcitx5 实时日志
journalctl --user -u fcitx5 -f

# RIME 状态自检
fcitx5-diagnose

# 重置 fcitx5 全部配置（先备份）
cp -r ~/.config/fcitx5 ~/.config/fcitx5.bak
rm -rf ~/.config/fcitx5 ~/.local/share/fcitx5
omarchy refresh fcitx5   # 如有此命令；否则手动重装上面所有包
```

常见问题：

- **只显示 keyboard-us**：profile 没把 rime 加进 Items。重写 `~/.config/fcitx5/profile` 后 `kill -9 $(pgrep fcitx5) && fcitx5 -d`。
- **打出来是繁体**：方案本身输出繁体，需切换到 `_simp` 后缀的方案（luna_pinyin_simp 等），或在用户目录用 OpenCC 简体补丁。
- **皮肤没生效**：检查 `~/.config/fcitx5/conf/classicui.conf` 是否被覆盖，`ls /usr/share/fcitx5/themes/` 确认皮肤目录存在。
- **某个应用调不出**：多半是应用自己的 IM 模块设置有问题，按第 5 节补环境变量。
- **能切到中文但打不出候选 / 方案一直显示 `.default`**：RIME 没真正部署，`~/.local/share/fcitx5/rime/build/` 缺失。修复：

  ```bash
  rm -rf ~/.local/share/fcitx5/rime/build
  pkill -9 fcitx5
  fcitx5 -d &
  # 等 ~3 秒后
  gdbus call --session --dest org.fcitx.Fcitx5 --object-path /rime \
    --method org.fcitx.Fcitx.Rime1.SetSchema "luna_pinyin_simp"
  # 验证
  ls ~/.local/share/fcitx5/rime/build/   # 应该有 *.prism.bin / *.schema.yaml
  gdbus call --session --dest org.fcitx.Fcitx5 --object-path /rime \
    --method org.fcitx.Fcitx.Rime1.GetCurrentSchema
  # ('luna_pinyin_simp',)
  ```

  如果 `build/` 重建后仍是空的，**手动触发部署**：

  ```bash
  cd ~/.local/share/fcitx5/rime
  rime_deployer --build .
  pkill -9 fcitx5 && fcitx5 -d
  ```

  触发条件一般是：第一次开机 `fcitx5 -d` 启动太早、RIME 还没初始化完；或手动 `kill` 之后没让 dbus 重新触发。修好后 Ctrl+Space 切中文，正常打字出候选。

---

## 9. 最终文件清单

```
~/.config/fcitx5/
├── conf/
│   └── classicui.conf              # Theme=ori
└── profile                         # 注册 rime + keyboard-us

~/.local/share/fcitx5/rime/
├── default.custom.yaml             # schema_list（luna_pinyin_simp 优先）
├── double_pinyin_mspy.custom.yaml  # __include emoji_suggestion:/patch
└── (可选) emoji_suggestion.custom.yaml

~/.config/hypr/
└── autostart.lua                   # o.launch_on_start("fcitx5 -d")
```

---

## 10. 方案切换速查

```bash
# 切到全拼简体（默认）
gdbus call --session --dest org.fcitx.Fcitx5 --object-path /rime \
  --method org.fcitx.Fcitx.Rime1.SetSchema "luna_pinyin_simp"

# 切到微软双拼
gdbus call --session --dest org.fcitx.Fcitx5 --object-path /rime \
  --method org.fcitx.Fcitx.Rime1.SetSchema "double_pinyin_mspy"

# 当前方案
gdbus call --session --dest org.fcitx.Fcitx5 --object-path /rime \
  --method org.fcitx.Fcitx.Rime1.GetCurrentSchema
```
