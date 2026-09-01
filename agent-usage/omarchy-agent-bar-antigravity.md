# Omarchy 顶部状态栏 AI Agent 面板集成 Antigravity 实时配额与自动化更新

---

## 1. 背景与需求

[Omarchy](https://omarchy.org/) 桌面环境（基于 Arch Linux + Hyprland + QuickShell）在顶部状态栏中内置了 AI Agents 使用量与额度监控面板，原生支持 Claude Code、Codex 等开发 Agent。

本实践旨在将 **Google Antigravity**（基于 Gemini 3.7 Flash / Pro）的真实使用额度无缝接入 Omarchy 状态栏面板，实现：
1. **精准展示配额**：展示 Gemini **5-Hour 滑动窗口剩余额度**及 **Weekly（周）额度**，并提供精确的重置倒计时。
2. **后台自动刷新**：无需手动打开面板或点击刷新，系统后台定时自动同步最新配额并在状态栏无感渲染。

---

## 2. 核心架构与原理解析

Omarchy 的 Agents 状态监控体系分为以下层次：

```
[ Antigravity Client / Language Server ]
                     │  (CDP WebSocket 端口 46387)
                     ▼
[ omarchy-agent-usage-antigravity 收集器 ]
                     │  (生成 JSON 记录)
                     ▼
[ ~/.local/state/omarchy/agents/usage/antigravity.json ]
                     │  (FileView watchChanges: true)
                     ▼
[ Omarchy 状态栏 / QuickShell Agents Panel ]
```

---

## 3. 踩坑排查与技术突破

### (1) 面板打开不自动触发更新
* **原因**：Omarchy 状态栏组件在启动与打开面板时，执行的是系统路径下的 `/usr/bin/omarchy-agent-usage-update`，该脚本严格扫描 `/usr/share/omarchy/bin/`。此前自定义收集脚本仅放置在 `~/.local/bin/`，导致后台调用时跳过了 Antigravity。
* **解决**：将自定义脚本软链接到系统级目录 `/usr/share/omarchy/bin/` 及 `/usr/bin/`。

### (2) 字符反推 Token 产生配额漂移
* **原因**：在多轮对话中，每一轮交互都需要将全部历史上下文打包装载给 Gemini API，使用本地日志字符数做静态或滑动累加计算与真实云端 Token 消耗存在偏差。
* **突破**：发现 Antigravity 运行时通过本地 Chrome DevTools Protocol（CDP 端口 46387）暴露了调试通道。通过该 WebSocket 执行 `getUserStatus()`，可**直接调用 Antigravity Language Server 获取 Google 官方云端返回的绝对真实剩余比例（`remainingFraction`）与重置时间戳（`resetTime`）**，实现 100% 官方精度。

---

## 4. 关键配置文件与实现

### ① 编写收集器脚本：`~/.local/bin/omarchy-agent-usage-antigravity`

脚本通过 Node.js 探针直连 CDP 端口抓取官方实时配额，并回退支持本地滑动统计：

```python
#!/usr/bin/python3
import datetime as dt
import json
import os
import shutil
import subprocess
from pathlib import Path
from typing import Any

AGENT_ID = "antigravity"
AGENT_NAME = "Antigravity"
MODEL_ID = "gemini-3.7-flash"

def find_node_bin() -> str:
    candidates = [
        "/home/guohh/.local/share/mise/installs/node/22.23.2/bin/node",
        "/usr/local/bin/node",
        "/usr/bin/node"
    ]
    for c in candidates:
        if os.path.isfile(c) and os.access(c, os.X_OK):
            return c
    found = shutil.which("node")
    return found if found else "node"

def probe_live_antigravity() -> dict[str, Any] | None:
    node_bin = find_node_bin()
    node_script = """
async function getLive() {
  try {
    const res = await fetch('http://127.0.0.1:46387/json/list');
    const targets = await res.json();
    const page = targets.find(t => t.type === 'page');
    if (!page) return null;

    return new Promise((resolve) => {
      const ws = new WebSocket(page.webSocketDebuggerUrl);
      const timer = setTimeout(() => { ws.close(); resolve(null); }, 2500);

      ws.onopen = () => {
        const expr = `(async () => {
          function findLsClient(obj, depth = 0, seen = new Set()) {
            if (!obj || depth > 6 || seen.has(obj)) return null;
            seen.add(obj);
            if (obj.getUserStatus && typeof obj.getUserStatus === 'function') return obj;
            if (obj._lsClient && typeof obj._lsClient.getUserStatus === 'function') return obj._lsClient;
            if (obj.lsClient && typeof obj.lsClient.getUserStatus === 'function') return obj.lsClient;
            for (const k of Object.keys(obj)) {
              try {
                const found = findLsClient(obj[k], depth + 1, seen);
                if (found) return found;
              } catch(e) {}
            }
            return null;
          }
          const root = document.getElementById('root');
          const rKey = root ? Object.keys(root).find(k => k.startsWith('__reactFiber') || k.startsWith('__reactContainer')) : null;
          const fiber = rKey ? root[rKey] : null;
          const foundClient = findLsClient(window) || (fiber ? findLsClient(fiber) : null);
          if (foundClient) {
            const res = await foundClient.getUserStatus({});
            return JSON.stringify(res, (k, v) => typeof v === 'bigint' ? v.toString() : v);
          }
          return null;
        })()`;
        ws.send(JSON.stringify({ id: 1, method: 'Runtime.evaluate', params: { expression: expr, awaitPromise: true, returnByValue: true } }));
      };

      ws.onmessage = (msg) => {
        clearTimeout(timer);
        try {
          const parsed = JSON.parse(msg.data);
          const rawStr = parsed.result?.result?.value;
          if (rawStr) {
            const data = JSON.parse(rawStr);
            const userStatus = data.userStatus || data;
            const models = userStatus.cascadeModelConfigData?.clientModelConfigs || [];
            const geminiFlash = models.find(m => m.modelId?.includes('gemini-3.7-flash') || m.label?.includes('Gemini 3.7 Flash')) || models[0];
            const tier = userStatus.userTier?.name || 'Google AI Pro';
            
            let geminiData = null;
            if (geminiFlash?.quotaInfo) {
              const resetSec = Number(geminiFlash.quotaInfo.resetTime?.seconds || 0);
              geminiData = {
                percent: Math.max(0.0, Math.min(1.0, geminiFlash.quotaInfo.remainingFraction)),
                resetsAt: resetSec ? new Date(resetSec * 1000).toISOString() : ''
              };
            }
            
            resolve({
              tierLabel: tier,
              gemini: geminiData
            });
          } else {
            resolve(null);
          }
        } catch(e) {
          resolve(null);
        }
        ws.close();
      };
      ws.onerror = () => { clearTimeout(timer); resolve(null); };
    });
  } catch(e) {
    return null;
  }
}
getLive().then(r => console.log(JSON.stringify(r)));
"""
    try:
        proc = subprocess.run([node_bin, "-e", node_script], capture_output=True, text=True, timeout=4)
        if proc.returncode == 0 and proc.stdout.strip():
            return json.loads(proc.stdout.strip())
    except Exception:
        pass
    return None
```

### ② 系统集成软链接
```bash
sudo ln -sf /home/guohh/.local/bin/omarchy-agent-usage-antigravity /usr/share/omarchy/bin/omarchy-agent-usage-antigravity
sudo ln -sf /home/guohh/.local/bin/omarchy-agent-usage-antigravity /usr/bin/omarchy-agent-usage-antigravity
```

### ③ 配置后台自动轮询服务 (Systemd User Timer)

创建服务文件 `~/.config/systemd/user/omarchy-agent-usage.service`：
```ini
[Unit]
Description=Update Omarchy AI Agent Usage metrics
After=graphical-session.target

[Service]
Type=oneshot
ExecStart=/usr/bin/omarchy-agent-usage-update
```

创建定时器文件 `~/.config/systemd/user/omarchy-agent-usage.timer`（每 30 秒执行一次）：
```ini
[Unit]
Description=Periodically update Omarchy AI Agent Usage metrics

[Timer]
OnBootSec=10s
OnUnitActiveSec=30s
AccuracySec=1s

[Install]
WantedBy=timers.target
```

启用并启动定时器：
```bash
systemctl --user daemon-reload
systemctl --user enable --now omarchy-agent-usage.timer
```

### ④ 状态栏面板刷新周期优化
将状态栏内置轮询间隔设为 30 秒：
```bash
omarchy bar set omarchy.agents refreshIntervalSec 30 --json
```

---

## 5. 效果与常用运维命令

* **手动即时更新一次**：
  ```bash
  omarchy-agent-usage-update
  ```
* **查看当前生成的指标数据**：
  ```bash
  cat ~/.local/state/omarchy/agents/usage/antigravity.json
  ```
* **查看后台定时器运行状态**：
  ```bash
  systemctl --user status omarchy-agent-usage.timer
  ```
