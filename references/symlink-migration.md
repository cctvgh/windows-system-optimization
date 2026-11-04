
# 符号链接迁移完整流程（C 盘大目录 → 其他盘）

## 适用场景

C 盘空间紧张，把大目录（应用数据、程序安装目录）迁到 D 盘等大容量盘。**不重装、不改快捷方式、文件关联照常**，程序无感知。

适合迁移的目录类型：

- `AppData\Roaming` / `AppData\Local` 下的应用数据（Tencent、kingsoft/WPS、baidu、Doubao 等）
- `AppData\Local\Programs` 下的用户级安装程序（Quark 等）
- `ProgramData` 下的共享数据（Anaconda3 等）
- `Program Files` / `Program Files (x86)` 下的程序目录（QGIS、WeMeet、Acrobat 等）

## 标准流程（五步）

### Step 1 停相关进程

```powershell
Get-Process | Where-Object { $_.ProcessName -match '<app关键词>' } | Stop-Process -Force
```

不停进程的后果：文件被占用 → 复制不完整或删源失败。迁移前逐个检查所有相关进程，**含后台服务进程**（如迁移 WPS 数据前停 wpscloudsvr）。

### Step 2 robocopy 复制

```powershell
robocopy $src $dst /E /COPYALL /R:1 /W:1 /MT:8 /NFL /NDL /NJH /NJS
```

- `/E` 含空目录；`/COPYALL` 复制权限等属性；`/MT:8` 多线程；`/R:1 /W:1` 失败快速跳过
- 大目录（7GB+）需几分钟，不要设过短超时

### Step 3 大小校验（关键安全门）

```powershell
$srcSize = (Get-ChildItem $src -Recurse -File -Force -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum
$dstSize = (Get-ChildItem $dst -Recurse -File -Force -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum
if ($dstSize -gt 0 -and [math]::Abs($srcSize - $dstSize) / [math]::Max($srcSize,1) -lt 0.05) {
    # 差异 < 5% 才允许删源
}
```

### Step 4 删源 + 建链接

```powershell
Remove-Item $src -Recurse -Force
cmd /c mklink /D $src $dst
```

**mklink 要求源路径不存在**，删源失败（文件被锁）时 mklink 报"当文件已存在时，无法创建该文件"。

### Step 5 验证

```powershell
$item = Get-Item $src -Force
$item.Attributes -band [System.IO.FileAttributes]::ReparsePoint   # True = 链接已建立
```

再启动对应程序确认功能正常（如 conda --version、WPS 打开文档）。

## 已知坑与处理

| 坑 | 现象 | 处理 |
| --- | --- | --- |
| 系统锁定文件删不掉 | Adobe 字体（*.ttf）、CMap 文件报"另一个程序正在使用" | D 盘副本已完整即视为迁移成功；C 盘残留几 MB 重启后手动补删 + 补 mklink；**不要回滚整个迁移** |
| mklink 源已存在 | "当文件已存在时，无法创建该文件" | 先确认源目录是否删干净；残留锁定文件按上一行处理 |
| 后台服务进程占用 | 复制完成但删源失败 | 迁移前 `Get-Process` 全量检查并停掉后台服务进程 |
| 权限继承 | ProgramData 下目录需管理员 | robocopy /COPYALL 保留 ACL，用管理员 PowerShell 执行 |

## 页面文件迁移（特殊：不走符号链接）

```powershell
# 1. 关闭自动管理
Get-CimInstance Win32_ComputerSystem | Set-CimInstance -Property @{AutomaticManagedPagefile=$false}
# 2. 移除 C 盘页面文件设置
Get-CimInstance Win32_PageFileSetting | Remove-CimInstance
# 3. 注册表指向 D 盘（0 0 = 系统管理大小）
Set-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management" -Name PagingFiles -Value "D:\pagefile.sys 0 0" -Type MultiString -Force
```

**重启后生效**：C:\pagefile.sys 自动删除，D:\pagefile.sys 自动创建。

## 回滚方法

1. 删除符号链接（只删链接不动目标数据）：`cmd /c rmdir "<原链接路径>"`
2. 把 D 盘内容 robocopy 回 C 盘原路径
3. 程序即恢复原状