# PCcolor — Windows 自动深浅色模式

按自定义时间自动切换 Windows 深色 / 浅色模式。基于 Windows **原生计划任务**实现：

- **零常驻**：不跑后台进程、不轮询时间，空闲内存占用为 0；到点由任务计划程序唤醒，脚本秒退。
- **无窗口**：切换由 `wscript.exe`（GUI 子系统，本身无控制台）以隐藏窗口方式执行，不弹终端、不打断工作。
- **首次设置即自动注册**：运行一次 `PCcolor settings`，填好时间即自动创建每日计划任务。
- **应用自动重启**：可配置一个应用列表，切换时自动关闭并重开这些应用，使其立刻应用新主题（解决部分应用需手动重启才刷新主题的问题）。

## 快速开始

1. 把整个 `PCcolor` 文件夹放到任意位置（例如 `D:\Tools\PCcolor`）。
2. 在该文件夹中打开终端（CMD 或 PowerShell）。
3. 输入：

   ```
   PCcolor settings
   ```

4. 按提示输入浅色 / 深色时间（24 小时制，如 `07:00` / `19:00`），确认后自动注册计划任务，并可选择立即应用当前时段应处的模式、配置自动重启的应用。

> Git Bash 中请使用 `./PCcolor.cmd settings`。
> 如果希望任意目录都能直接敲 `PCcolor`，可把该文件夹加入用户 PATH（可选，非必需）。

## 命令一览

| 命令 | 说明 |
| --- | --- |
| `PCcolor settings` | 进入设置；首次运行自动注册计划任务 |
| `PCcolor status` | 查看当前主题、配置、计划任务状态与下次切换时间 |
| `PCcolor switch dark` | 立即切换到深色 |
| `PCcolor switch light` | 立即切换到浅色 |
| `PCcolor uninstall` | 卸载计划任务（停止自动切换） |
| `PCcolor help` | 帮助 |

设置菜单中还可：修改时间、立即切换、重新注册、卸载、**管理自动重启的应用列表（[6]）**。

## 自动重启应用（解决“应用不刷新、要手动重开”）

切换主题时，绝大多数系统组件和应用会通过广播即时刷新；少数应用（尤其部分 Win32 程序）不会。把这些应用加入列表，切换时会自动关闭并重新打开它们：

```
PCcolor settings  ->  [6] 管理自动重启的应用列表
```

- `[1] 从当前运行的应用中添加`：列出当前带窗口的应用，输入编号即可（自动记录进程名和程序路径）。
- `[2] 手动输入添加`：输入进程名（如 `notepad`，可不带 `.exe`）和程序完整路径（可留空，切换时自动解析）。
- `[3] 移除应用`：按编号移除。

也可直接编辑 `scripts\apps.list`，每行一个应用：

```
notepad|C:\Windows\System32\notepad.exe
msedge|C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe
```

> **注意**：被配置的应用会在切换时被关闭（先优雅关闭，3 秒后仍未退出则强制结束），**未保存的内容可能丢失**，请谨慎添加。
> 任务栏 / 开始菜单 / 设置等系统组件随广播自动刷新，无需加入列表。

## 工作原理

1. 注册两个**每日**计划任务（仅当前用户、交互式登录、Limited 权限，无需管理员、不弹 UAC）：
   - `PCcolor-Light`：每天浅色时间点执行
   - `PCcolor-Dark`：每天深色时间点执行
2. 触发时由 `wscript.exe`（无控制台）静默启动 PowerShell（隐藏窗口）执行 `scripts\apply-theme.ps1`：
   - 写入 `HKCU\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize` 下的 `AppsUseLightTheme` 与 `SystemUsesLightTheme`（0 = 深色，1 = 浅色）；
   - 广播 `WM_SETTINGCHANGE (ImmersiveColorSet)`，任务栏、开始菜单和大多数应用即时刷新，无需重启资源管理器；
   - 按 `scripts\apps.list` 重启配置的应用，使其打开时即为新主题；
   - 全过程写入 `scripts\logs\switch.log`，可在 `PCcolor status` 查看最近日志。
3. 任务设置：允许电池供电运行、错过时间点在唤醒后补做（StartWhenAvailable）、执行超时 5 分钟。

## 目录结构

```
PCcolor/
├─ PCcolor.cmd               命令入口（settings / status / switch / uninstall / help）
├─ run-hidden.vbs            计划任务无窗口启动器
├─ README.md
└─ scripts/
   ├─ menu.ps1               设置向导 + 交互菜单 + 状态 + 应用管理
   ├─ apply-theme.ps1        切换核心（注册表 + 广播 + 应用重启）
   ├─ register-tasks.ps1     计划任务注册 / 卸载
   ├─ apps.list              自动重启应用列表（可编辑）
   ├─ config.json            用户配置（首次设置生成）
   └─ logs/switch.log        切换日志（自动生成）
```

## 常见问题

- **部分 Win32 应用不即时刷新**：用菜单 `[6]` 把它加入自动重启列表。
- **笔记本休眠错过时间点**：唤醒后自动补做一次。
- **移动了文件夹**：重新运行 `PCcolor settings` → 选 `[4]` 重新注册计划任务（旧路径失效）。
- **想彻底移除**：`PCcolor uninstall`，然后删除文件夹；计划任务中不会留下残余。
- **公司电脑注册任务被策略拒绝**：按报错提示处理；临时可用 `PCcolor switch` 手动切换。
- **多用户账户**：每个 Windows 账户需各自执行一次 `PCcolor settings`（任务按用户注册）。

## 安全与隐私

仅向当前用户注册表（HKCU）写入两个主题值、广播一条系统设置变更消息、按需重启用户自己配置的应用；**无网络访问、无数据上传、无第三方依赖**。全部为脚本文件，可自行审计。
