
# 企业安全软件约束与受保护服务识别

## 一、绝不能动的软件清单（企业管控环境）

按进程/服务名识别。同一厂商通常有"客户端 + 守护 + 托盘 + 更新器"全家桶，**全部不能动**。

| 厂商/类别 | 典型进程/服务/自启项 | 说明 |
| --- | --- | --- |
| 天融信终端安全 | TopsecClient、TopsecDaemon、sysdiag(TopsecTray)、Topsec PopBlock | 杀毒/端点安全，关掉 = 杀毒失效 |
| 天融信桌面管理 | TopDesk Tray、tdsvc、Topdesk Client Updater | 终端管控 |
| 深信服零信任 | aTrustService、aTrustAgent | 内网接入通道，关掉 = 无法接入内网 |
| 堡垒机 | bh_local、Bastion 相关 | 运维审计合规 |
| 电子文件安全管理系统 | SLClient、SL_ASSIST、SL_MONITOR、PluginControl | 企业级文档管控 |
| SDP 安全接入 | SdpUpdate10、C:\Sdp\SecAccess_* | 安全接入更新 |
| 网管客户端 | Launcher (NetNumen)、netnumen 目录 | 网络管理 |
| 输入法 | ctfmon | 系统输入服务，关掉输入法异常 |

### 识别技巧

- 服务/自启项的可执行文件路径含 `Topsec`、`Sangfor`、`Sdp`、`SLClient`、`Bastion`、`netnumen` → 默认不能动
- 拿不准的用 `(Get-Item <exe>).VersionInfo.ProductName` 反查（如 SLClient 显示"智能电子文件安全管理系统"）
- 计划任务中安全厂商的任务（如 Topsec PopBlock、Topdesk Client Updater）同样不能禁用
- 启动文件夹中的安全系统客户端（如 SLClient.exe.lnk）不能删

## 二、受安全策略保护的服务（改不动 ≠ 权限问题）

### 现象

对某服务执行以下操作**全部失败**（Access Denied / 拒绝访问），即使以管理员运行：

- 用服务管理命令把启动类型改为「禁用」→ 拒绝访问
- 用旧式服务配置命令改启动类型 → 失败 5：拒绝访问
- 用 WMI 服务管理类改启动模式 → ReturnValue=2（Access Denied）
- 直接写注册表 `HKLM:\SYSTEM\CurrentControlSet\Services\<name>` 的启动值 → UnauthorizedAccessException
- 查看服务安全描述符显示 ACL 正常（Administrators 有 FullControl）——但就是写不进

### 结论与处理

这是**安全软件的注册表保护**（如天融信 ESEndpoint 对特定服务键的防护），不是普通权限问题：

1. 快速验证 2~3 种方式（服务管理命令、服务配置命令、注册表写入）确认是"保护"而非偶发失败
2. **停止尝试提权 workaround**：获取所有权、修改 ACL、SYSTEM 提权均无效，浪费时间且可能触发安全告警
3. 报告中标注：该服务被安全策略保护，需 IT 管理员在管控平台后台解除保护，或卸载对应客户端
4. 同厂商/同套件其他服务通常不受保护，可正常禁用

### 已知案例

- 云桌面客户端主服务 `clink_service`（C:\Program Files (x86)\CloudDesktop\clink\64\clink_service.exe）：被保护，StartType 始终 Auto Start 改不动
- 同套件不受保护、可正常禁用的：clinkte_service、clipa、cloudshare_service、BalloonService、CloudPrinterClientService、AppMarketSvc、cloudbase-init

## 三、相关注意

- 部分服务 `Stop-Service` 在当前会话停不掉（仍 Running），但 StartType 已改为 Disabled——**重启后即不再启动**，属正常现象，不要反复强停
- 安全软件的弹窗拦截、更新任务看似"多余"，但属于管控策略一部分，一律保留