---
name: windows-system-optimization
description: "Whole-machine Windows optimization for boot speed, disk space, and runtime performance. Covers multi-source startup item audit (registry Run keys, startup folders, logon scheduled tasks, auto services) with safe categorization, cache cleanup, symlink migration of large directories to another drive, pagefile relocation, visual/TCP/event-log tuning, defragmentation, sfc verification, and activation status check. Built for enterprise-managed machines: security/compliance software is never touched and uncertain items are confirmed by the user."
name_cn: Windows系统优化
description_cn: Windows整机优化：开机自启项审计清理、缓存清理、符号链接迁移大目录扩容C盘、页面文件迁移、视觉效果/TCP/碎片整理调优、激活状态检查；企业安全管控环境下安全软件绝不触碰，拿不准的项交用户确认。
---

# Windows 系统优化（开机加速 / 磁盘扩容 / 运行调优）

## 技能定位与边界

覆盖 Windows（Win7~Win11 / Windows Server）**整机优化**的三类高频任务：

1. **开机加速** —— 自启项全面审计、分类、清理
2. **磁盘空间优化** —— 缓存清理 + 大目录符号链接迁移到其他盘 + 页面文件迁移
3. **运行性能调优** —— 视觉效果、TCP、事件日志、碎片整理、系统文件修复、内存释放

**与 teleagent-self-repair 的边界**：本技能针对整机 Windows 层面；TeleAgent 应用自身的检修（其缓存/数据库/配置文件）走 teleagent-self-repair。用户说"电脑卡/C盘满"时，若根因在 TeleAgent 自身文件，优先用 teleagent-self-repair。

**适用环境**：企业安全管控机器（装有杀毒/终端管控/零信任 VPN/堡垒机等）。此类环境有额外红线，见下。

## 核心红线（必须遵守）

1. **安全/合规/终端管控类软件绝不触碰**：杀毒客户端及其守护/托盘/弹窗拦截、零信任 VPN、堡垒机、电子文件安全管理系统、SDP 安全接入、网管客户端、输入法进程（ctfmon）。误关会导致杀毒失效、内网无法接入、违反企业安全策略。识别清单见 `references/enterprise-security-constraints.md`。
2. **拿不准的项交由用户决定**：日常软件（聊天、网盘、AI 助手、搜索工具、WPS 云服务等）是否保留自启，用分组多选题让用户勾选，不替用户判断。
3. **改注册表前先备份**：`reg export` 保存 Run 键到临时目录，报告中告知备份位置与恢复方法。
4. **迁移不卸载**：程序/数据迁移用符号链接，软件本体、快捷方式、文件关联全部照常，用户随时可手动打开。
5. **系统级修改注明"重启后生效"**（服务禁用、页面文件迁移、视觉效果），提醒用户确认无未保存任务后再重启。

## 通用工作流

```
扫描（只读）→ 分类（不能动 / 明显多余 / 拿不准）→ 用户确认 → 备份 → 执行 → 验证 → 报告
```

---

## 模块一：开机自启项审计与清理

### 1. 数据源（全部要扫，缺一不可）

| 数据源 | 采集方式 |
| --- | --- |
| HKLM Run / HKLM Wow6432Node Run | `Get-ItemProperty "HKLM:\Software\Microsoft\Windows\CurrentVersion\Run"` 及 Wow6432Node 路径 |
| HKCU Run / RunOnce（HKLM+HKCU） | 同上，HKCU 路径 |
| 启动文件夹 | `$env:ProgramData\Microsoft\Windows\Start Menu\Programs\Startup` 与 `$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup` |
| 登录/开机计划任务 | `Get-ScheduledTask \| Where-Object { $_.State -ne 'Disabled' -and $_.Triggers.CimClass.CimClassName -match 'LogonTrigger\|BootTrigger' }` |
| 自启服务 | `Get-CimInstance Win32_Service \| Where-Object { $_.StartMode -eq 'Auto' -and $_.PathName -notmatch 'system32\|Windows' }` |
| StartupApproved（含历史残留） | `HKLM/HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\StartupApproved\Run` |

未知项不要凭名字猜：用 `(Get-Item <exe>).VersionInfo`（CompanyName/ProductName）和注册表 Uninstall 项反查归属。

### 2. 分类规则

- **不能动**：安全软件全家桶（红线 1）、ctfmon、系统核心服务。
- **明显多余**（可直接关，报告中说明理由）：
  - 古董版预加载器（如 Acrobat 9 的 Speed Launcher / Acrotray）
  - 已被系统内置取代的驱动监控（如 NUSB3MON）
  - 浏览器后台预加载（如 MicrosoftEdgeAutoLaunch）
  - 更新器自启任务（App 自己能检查更新，如 QianwenUpdater、GoogleUpdater）
  - 云主机初始化服务（cloudbase-init，早已执行完毕且 Stopped）
  - 每次登录都跑的清理类任务（如 CCleaner 登录任务 / BugReport / Update）
- **拿不准**：一切日常软件自启 + 后台服务（云电脑客户端套件、虚拟机服务、更新服务）→ 分组多选题问用户；服务类可提供"禁用"与"改 Manual（用时自启）"两档。

### 3. 执行

- Run 键：`Remove-ItemProperty`（先 `reg export` 备份 HKLM / HKLM Wow64 / HKCU 三个 Run 键）
- 计划任务：`Disable-ScheduledTask`（不删除，可随时 Enable 恢复）
- 服务：通过服务管理命令设置启动类型为「禁用」（彻底不用）或「手动」（VMware/更新类，用时自启）
- 残留清理：StartupApproved 中 Run 键已不存在的孤儿项、已移除自启项的 Prefetch 文件、IconCache.db、缩略图缓存

### 4. 关键坑：受安全策略保护的服务

某些服务被安全软件锁定注册表：无论通过服务管理命令、WMI 还是直接写注册表服务键项，一律返回 Access Denied——即使以管理员运行、查看服务安全描述符显示 ACL 正常。

**识别后立即停止尝试**（获取所有权、修改 ACL、SYSTEM 提权均无效，浪费时间且可能触发安全告警），报告中标注"被安全策略保护，需 IT 管理员在管控后台解除或卸载对应客户端"。同套件其他服务通常不受保护，可正常禁用。详见 `references/enterprise-security-constraints.md`。

---

## 模块二：磁盘空间优化

### 1. 扫描顺序

`Get-Volume` 卷概况 → C 盘根目录各文件夹 → `C:\Users\<user>` 子目录 → AppData\Local 与 Roaming 大文件夹 → ProgramData → Program Files (x86) / Program Files → 系统文件（hiberfil/pagefile/swapfile）→ 开发缓存（pip/conda/npm）→ Windows\Installer。

### 2. 可清理项

Windows\Temp、用户 Temp、SoftwareDistribution\Download、CBS 日志、WER、pip cache、conda pkgs 缓存（`conda clean` 在系统级安装下可能清不彻底，可直接删 pkgs 内压缩包与解压目录，需要时自动重新下载）。

### 3. 符号链接迁移（核心方法）

要点（完整流程与坑见 `references/symlink-migration.md`）：

1. **先停相关进程**（含后台服务进程，如迁移 WPS 数据前停 wpscloudsvr）
2. `robocopy <src> <dst> /E /COPYALL /R:1 /W:1 /MT:8`
3. **大小校验**：源/目标总字节数差异 <5% 才允许删源
4. 删源 → `cmd /c mklink /D <src> <dst>`（mklink 要求源路径不存在，否则报"文件已存在"）
5. 验证 ReparsePoint 属性 + 程序启动测试

**坑**：系统锁定文件（字体/CMap 等）删不掉时，D 盘副本已完整即视为迁移成功，C 盘残留几 MB 重启后补删补链接，**不要回滚**。

### 4. 页面文件迁移

`Win32_ComputerSystem.AutomaticManagedPagefile=false` → 移除 C 盘 `Win32_PageFileSetting` → 注册表 `PagingFiles` 设 `D:\pagefile.sys 0 0`。重启后生效，C 盘 pagefile.sys 自动删除。

---

## 模块三：运行性能调优

| 项 | 操作 | 说明 |
| --- | --- | --- |
| 视觉效果 | `VisualEffectsSetting=2`（最佳性能）、`MinAnimate=0`、`MenuShowDelay=0` | 界面响应变快 |
| 启动延迟 | `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\StartupDelayInMSec=0` | 桌面出现后立即可操作 |
| 内存大户 | 结束后台常驻大户进程（如云 AI 助手多进程占 400MB+） | **必须用户确认后结束** |
| TCP | `netsh interface tcp set global ecncapability=enabled`、`timestamps=enabled`、TIME_WAIT 降至 30s、清 DNS 缓存 | 吞吐与连接回收改善 |
| 事件日志 | `wevtutil cl <log>` 清理 + `wevtutil sl <log> /ms:8388608` 限额 | 防日志膨胀 |
| 系统文件 | `sfc /scannow` | **控制台输出可能乱码，用 `C:\Windows\Logs\CBS\CBS.log` 中 `[SR]` 行确认结果**（如 Verify complete / Repairing N components） |
| 碎片整理 | 先 `defrag X: /A /V` 分析，碎片率高再整理 | 见下 |

### 碎片整理要点

- 虚拟机磁盘（VirtIO 等）同样适用；碎片率高（如 53%）会显著拖慢文件读取
- 后台运行 `defrag X: /U /V` 并轮询日志，不要同步等待（可能 30 分钟以上）
- **两个盘并行整理会争磁盘 I/O 互相拖慢**——建议先系统盘、后数据盘
- 整理后复查并报告前后碎片率对比

### 激活状态检查（用户问激活/密钥时）

先查实际状态再回答：`slmgr /dli`、`slmgr /xpr`、`Get-CimInstance Win32_OperatingSystem`。企业机器常已通过 KMS 激活（VOLUME_KMSCLIENT channel，内网 KMS 服务器自动续期），根本无需操作。**只提供合规途径**（单位 IT/KMS、机箱 COA 标签、微软客服），不提供破解工具与密钥。

---

## 模块四：验证与报告

### 验证

- 自启项：重新枚举所有数据源，剩余项应全部属于"不能动"或用户保留
- 服务：确认 StartType 已变；**StartType=Disabled 但 Status=Running 是本次会话残留，重启后不再启动，属正常**
- 迁移：ReparsePoint 属性 + 程序启动测试
- 空间/内存：前后对比 `Get-Volume` 与 `Win32_OperatingSystem.FreePhysicalMemory`

### 报告格式（用户要"关了哪些、预计快多少"）

1. **关了哪些**：分表列出（注册表自启项 / 计划任务 / 服务），标注"明显多余"或"你确认关闭"及理由
2. **没动的**：安全/系统类清单 + 原因（让用户放心）
3. **未能完全关闭的**：受保护服务单独列出 + 建议（找 IT）
4. **预计快多少**：量化估算（关 N 个聊天/AI 软件省几秒 + 释放几百 MB；禁用 N 个服务省几秒），注明重启后生效
5. **恢复方式**：备份位置、恢复单个自启项的方法

## 支持文件

- `references/enterprise-security-constraints.md` —— 企业安全软件识别清单、受保护服务识别与处理
- `references/symlink-migration.md` —— 符号链接迁移完整流程、命令模板、坑与回滚