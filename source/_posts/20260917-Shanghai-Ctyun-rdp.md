---
title: 上海电信云电脑如何使用RDP连接
published: 2026-09-17
tags: [Markdown, 上海, 云电脑, rdp]
category: 云电脑
draft: false
---

[](#p-22305875-h-1)问题点
----------------------

领到免费云电脑后，使用tailscale进行虚拟组网，开启远程连接，会发现无法连接到这个电脑。

[](#p-22305875-h-2)原因和解决方案
--------------------------

Windows Server 当前没有可供 RDP 使用的 Remote Desktop 自签名证书。  
**解决方案**

[](#p-22305875-step-1-rdp-3)Step 1：恢复 RDP 自签名证书存储配置
---------------------------------------------------

以管理员身份打开 PowerShell，执行：

```
New-ItemProperty `
  -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations' `
  -Name 'SelfSignedCertStore' `
  -PropertyType String `
  -Value 'Remote Desktop' `
  -Force 
```

验证：

```
Get-ItemProperty `
  'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations' `
  -Name SelfSignedCertStore 
```

期望：

```null
SelfSignedCertStore : Remote Desktop

```

* * *

[](#p-22305875-step-2-rdp-4)Step 2：启动 RDP 证书生成相关服务
--------------------------------------------------

执行：

```
Start-Service CryptSvc
Start-Service KeyIso
Start-Service SessionEnv
Start-Service CertPropSvc 
```

然后检查：

```
Get-Service CertPropSvc,KeyIso,CryptSvc,SessionEnv |
Format-Table Name,Status,StartType 
```

这些服务的 Startup Type 不一定全部是 Automatic。

关键点是：

> 它们应该能够正常启动，而不是 Disabled 或启动时报错。

* * *

[](#p-22305875-step-3-rdp-5)Step 3：确认基础 RDP 配置
----------------------------------------------

执行：

```
Set-ItemProperty `
  -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server' `
  -Name 'fDenyTSConnections' `
  -Value 0 
```

执行：

```
Set-ItemProperty `
  -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' `
  -Name 'fEnableWinStation' `
  -Value 1 
```

确认端口：

```
Get-ItemProperty `
  'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' `
  -Name fEnableWinStation,PortNumber 
```

期望：

```null
fEnableWinStation : 1
PortNumber        : 3389

```

* * *

[](#p-22305875-step-4-windows-rdp-6)Step 4：启用 Windows 防火墙 RDP 规则
----------------------------------------------------------------

执行：

```
Get-NetFirewallRule |
Where-Object {$_.Name -like "RemoteDesktop*"} |
Enable-NetFirewallRule 
```

**然后重启电脑就能正常连接了**