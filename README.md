# iVentoy 批量部署 Windows 11

自用操作单：UEFI/GPT 客户机 + 第三方 DHCP + Win11 Pro 全自动装机。

```
加电 → PXE → 菜单 5s 自动进 ISO → 擦盘装系统 → 自动登录 Administrator
     → 首次登录拉 install-drivers.cmd → 下载 tool 到 C:\tool → 弹出驱动总裁
```

驱动总裁保留图形界面，要现场点一下，不是全自动。

## 服务端准备

iVentoy **1.0.43+**，解压路径不要有中文/空格。

```
<iVentoy>\
├── iso\<win11>.iso
├── user\scripts\unattend.xml      ← 仓库 unattend.xml（UTF-8 with BOM）
└── user\deploy\
    ├── install-drivers.cmd        ← 仓库同名文件（纯 ASCII）
    ├── tool_files.txt             ← 仓库里是空占位，部署时生成
    └── tool\                      ← 驱动/工具目录
```

`tool_files.txt` 每次部署前在 `tool\` 目录生成（结果必须 ASCII + CRLF）：

```powershell
$root = (Get-Location).Path
Get-ChildItem -Recurse -File |
  ForEach-Object { $_.FullName.Substring($root.Length + 1).Replace('\','/') } |
  Where-Object { $_ -match '^[A-Za-z0-9._/\-]+$' } |
  Sort-Object | Set-Content -Encoding ASCII ..\tool_files.txt
```

`tool\` 里增删过文件就要重跑，否则客户机漏文件（日志 `MISSING or EMPTY`）。
文件名只用 ASCII——中文/空格过不了 cmd.exe 的 OEM 代码页。

## 改 unattend.xml

搜 `EDIT ME`，三处：

| 位置 | 改成 |
|---|---|
| `/IMAGE/NAME` | `Windows 11 Pro`（换 ISO 后 `dism /Get-WimInfo` 照抄 Name） |
| `AutoLogon` 的 `<Value>` | 密码 |
| `AdministratorPassword` 的 `<Value>` | 同一个密码，两处必须一致 |

- 选版本用 **NAME，不用 INDEX**：多版本 ISO 里索引 1 是家庭版，写错不报错、静默装错。
- 不要删 `AcceptEula`、`ProductKey`；不要填真实 MAK / 零售密钥。
- 不要加 `net user Administrator /active:yes`，会让 OOBE 失败；启用靠 `AutoLogon`。
- 选盘用 `$$VT_WINDOWS_DISK_CLOSEST_200$$`，`DiskConfiguration` 和 `InstallTo` 两处都要写，不要写死 `0`。

⚠️ **改完必须重启 iVentoy**：脚本缓存在内存里，只在启动时读一次，不重启会继续下发旧内容且不报错。

## 界面配置（26000）

**参数设置**：菜单默认超时 `5`、DHCP 模式 `ProxyNet`、勾上 ByPass HW Check + ByPass NRO。

**镜像管理**（选中 ISO）：设为默认启动文件；新增自动安装脚本 `unattend.xml`，「默认脚本编号」设为 1，
「脚本选择超时」设非 0；注入文件留空。

防火墙放通 **16000**。

## 装机

1. `unattend.xml` 改完、iVentoy 重启过。
2. UEFI 启动、关 CSM；Secure Boot 要么在 BIOS 里关掉，要么按 iVentoy 的 Standard/ByPass 模式处理。
3. **拔掉所有 U 盘和移动硬盘**——选盘变量不排除 USB 盘，会被擦掉；首测再拔掉数据盘。
4. 客户机 PXE 启动，等它自己走完。

## 验收（每批第一台）

| 项 | 怎么查 |
|---|---|
| 分区 GPT + EFI + MSR | `diskpart` → `list partition` |
| 版本是 Pro | `winver` |
| Administrator 已启用 | `net user Administrator` |
| 自动登录生效 | 重启后直接进桌面 |
| 驱动总裁被拉起 | `C:\Windows\Temp\install-drivers.log` |
| 浏览器主页/搜索没被改 | 打开浏览器看一眼 |

计算机名是 `DESKTOP-XXXXXX` 随机名，正常。

## 排错

| 现象 | 处理 |
|---|---|
| `PXE-E51` / 拿不到 IP | 第三方 DHCP 不响应 PXE → 用 iVentoy 内置 DHCP；开了 Snooping 就把 iVentoy 端口设 trusted |
| `PXE-E53` / 没有 bootfile | DHCP 模式切 `ProxyNet`（或用 `External` 配 066/067） |
| 菜单不自动走 | 菜单默认超时 = 0 |
| 停在「选择自动安装脚本」 | 脚本选择超时 = 0 |
| 弹出分区界面 / 先问语言键盘 | `unattend.xml` 没生效：路径、默认脚本编号、是否放在 `user\scripts` |
| 「无法分析或处理无人参与应答文件」 | BOM 丢了，另存为 UTF-8 with BOM |
| 卡在选版本 / 装成家庭版 | `/IMAGE/NAME` 与镜像里的 Name 不符，或用了 INDEX |
| 停在「产品密钥」页 | 缺 `ProductKey` |
| 「缺少计算机所需的介质驱动程序」 | 缺的是**网卡**驱动，Hyper-V 测不出来；需要注入驱动（方案在 git 历史里） |
| 改了 xml 但客户端行为没变 | 没重启 iVentoy |
| 自动登录没生效、停在锁屏 | `AutoLogon` 与 `AdministratorPassword` 密码不一致 |
| 没弹驱动总裁 | 看 `install-drivers.log`；没有这个文件 → 16000 不通或 `user\deploy\` 缺文件 |
| 日志出现 `MISSING or EMPTY` | `tool` 加过文件但没重新生成清单 |
| 驱动总裁装不出驱动 | 它要联公网，客户机只通局域网就会失败 |

## 注意

- 密码 `root123` 明文写在 xml 里，`user/` 又走 HTTP 对外开放，**只适合实验环境**；真部署前改密码。
- 免费版最多 20 台、禁止商用。
- 确认客户机拿到的是新版：回传日志 `log\client\<ip>.zip` 里 `ventoy.log` 的 `len:` 应等于磁盘上 `unattend.xml` 的字节数。
- `Windows.old` 还在，说明 `<WillWipeDisk>` 可能没按预期擦盘；给有数据的机器装机前先确认。
- 日志时间戳不精确（`%time%` 在循环里只展开一次），故意没修。

更细的说明（清单规则、DrvCeo 参数、start.bat）见 [user/deploy/README.md](user/deploy/README.md)。
