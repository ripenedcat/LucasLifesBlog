+++
author = "Lucas Huang"
title= "排查 Azure Update Manager 中常见的 Windows Update HRESULT 错误"
date= '2026-02-21T09:30:00+08:00'
draft= false
tags = [
    "Azure Update Manager",
    "HRESULT",
    "Windows Update Client"
]
categories= ["Azure"]
image = "cover.png"
+++

在使用 Azure Update Manager 管理服务器机队的更新时，您可能偶尔会遇到修补或评估失败的情况。通常，这些故障会在日志、Azure 门户 UI 或 Azure Resource Graph 查询输出中以难以理解的 `HRESULT` 错误代码形式出现。

例如，您可能会看到如下错误字符串：
```
Assessment failed due to this reason: '['Windows update API failed to assess the machine for available updates. Error:Exception from HRESULT: 0x8024402C, Hresult:0'    at UpdateManagementAction.concrete.UpdateAssessment.GetAvailableUpdates()
   at UpdateManagementAction.concrete.AssessmentAction.Assess(UpdateManagementResultSummary updateActionResultSummary)].'
'83 errors reported. The latest 3 errors are shared in details. To view all errors, review this log file on the machine:[C:\WindowsAzure\Logs\Plugins\Microsoft.CPlat.Core.WindowsPatchExtension\1.5.54]
```
![AUM Failure Error Message](aum-failure-error-message.png)

这些是机器上底层 Windows Update 客户端返回的标准错误代码，每个代码都指向特定的失败原因。在本文中，我们将详细分析最常见的 `HRESULT` 错误，解释它们的含义，并提供可行的解决步骤。

<!--more-->

## 黄金法则：可以在本地进行打补丁吗？
在深入研究特定的错误代码之前，建立一个基准非常重要：**是否可以使用“控制面板/设置”中的本地 Windows Update 客户端手动给机器打补丁？**

如果本地打补丁失败，则问题出在 Windows 操作系统、本地网络或 Windows Update 代理本身 —— 而不是 Azure Update Manager。在这些情况下，您正在处理的是核心 Windows 操作系统问题。

如果您陷入困境，查看 `C:\Windows\WindowsUpdate.log` 文件始终是进行深入调查的良好起点。


## 解析常见的 HRESULT 错误

以下是最常遇到的 `HRESULT` 代码及其修复方法的详细说明。

### 0x80070005 (E_ACCESSDENIED)
**含义：** 一般拒绝访问错误。

**修复方法：** 这通常由不正确的 Windows Update 设置、`%WinDir%\SoftwareDistribution` 文件夹中的文件权限错误或系统驱动器 (C:) 上的磁盘空间不足引起。释放一些空间并验证系统权限。

### 0x80070643 (ERROR_INSTALL_FAILURE)
**含义：** 一般安装失败。

**修复方法：** 最近，这与尝试安装特定安全更新（如 KB5034439、KB5034440 或 KB5034441）时 **Windows 恢复环境 (WinRE) 分区** 上的磁盘空间不足密切相关。您可能需要手动调整 WinRE 分区的大小以使更新成功安装。

### 0x80072EE2 (WININET_E_TIMEOUT)
**含义：** 操作超时。

**修复方法：** 这是一个典型的网络连接问题。机器无法与内部 WSUS 服务器、公共 Windows Update 站点或配置的代理进行通信。检查防火墙规则并运行网络跟踪以查看流量在哪里被丢弃。

### 0x80072EE6 (WININET_E_UNRECOGNIZED_SCHEME)
**含义：** URL 未使用可识别的协议。

**修复方法：** 这通常发生在组策略或注册表中定义了 WSUS 服务器，但缺少 `http://` 或 `https://` 前缀时。
检查 `HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\WUServer`。如果值只是 `wsus.contoso.com`，请将其更改为 `http://wsus.contoso.com`。

### 0x80072EFE (WININET_E_CONNECTION_ABORTED)
**含义：** 与服务器的连接异常终止。

**修复方法：** 这意味着连接被防火墙、代理或 Windows Update 服务器本身强制重置。检查网络安全设备以确保到 Windows Update 端点的流量未被阻止或进行不当的检查。

### 0x80072F8F (WININET_E_DECODING_FAILED)
**含义：** 内容解码失败（加密/TLS 故障）。

**修复方法：** 这通常由于以下三个原因之一发生：
1. Windows Server 时钟未同步。验证系统时间。
2. VM 无法访问 `https://ctldl.windowsupdate.com`。
3. 存在 TLS 或密码套件不匹配。检查 `HKLM\SYSTEM\CurrentControlSet\Control\Cryptography\Configuration\Local\SSL\00010002\Functions` 中的密码套件，并确保它们符合默认的 Windows 要求。

### 0x800f0982 (PSFX_E_MATCHING_COMPONENT_NOT_FOUND)
**含义：** 无法识别用于混合(hydration)的匹配组件。

**修复方法：** 这通常与语言包不匹配或已知的服务堆栈问题有关（例如，在特定的补丁星期二影响非美国英语的 Windows Server 2019 机器）。请检查您的特定操作系统版本的微软已知发布健康问题。

### 0x80240009 (WUEOPERATIONINPROGRESS)
**含义：** 另一个冲突的操作正在进行中。

**修复方法：** 当 Azure Update Manager 触发时，机器已经忙于安装更新。您只需等待现有的修补操作完成即可。确保没有重叠的更新计划。

### 0x8024000E
**含义：** Windows Update Agent 在更新的 XML 数据中发现无效信息。

**修复方法：** 虽然听起来像是 XML 格式问题，但这通常指向 VM 上的严重资源耗尽。检查 VM 是否内存不足或存储驱动器是否已完全存满。

### 0x80240016 (WU_E_INSTALL_NOT_ALLOWED)
**含义：** 正在进行另一个安装，或系统正等待强制重启。

**修复方法：** 类似于 0x80240009，等待当前安装完成。如果系统需要因先前的补丁重启，请重启服务器然后重试。

### 0x8024001E (WU_E_SERVICE_STOP)
**含义：** 操作未完成，因为服务或系统正在关闭。

**修复方法：** 在补丁作业运行时，Windows Update 服务（或整个服务器）已停止。确保在 `services.msc` 中运行 `Windows Update` 服务，然后重试。

### 0x8024002E (WU_E_WU_DISABLED)
**含义：** 不允许访问非托管服务器。

**修复方法：** Windows Update 客户端服务已被策略禁用。检查注册表项 `HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\DisableWindowsUpdateAccess`。如果将其设置为 `1`，请将其更改为 `0`（最好通过组策略进行）。

### 0x80240438 (WU_E_PT_ENDPOINT_UNREACHABLE)
**含义：** 没有到端点的路由或网络连接。

**修复方法：** 网络配置错误。通常，机器被配置为使用 WSUS 服务器，但实际上没有有效的网络路由来访问它。调查路由表和 VNet 配置。

### 0x8024401C
**含义：** 服务器等待请求超时 (HTTP 408)。

**修复方法：** 这通常意味着您的内部 WSUS 服务器过载并正在断开连接。您将需要对 WSUS 服务器执行维护（例如，清理过时的更新、增加 IIS 应用程序池内存）。

### 0x8024402C
**含义：** 无法解析代理服务器或目标服务器名称。

**修复方法：** DNS 或代理问题。
*   运行 `netsh winhttp show proxy` 查看是否设置了错误的代理。
*   检查配置的 WSUS 服务器名称是否可以通过 DNS 实际解析 (`Resolve-DnsName <wsus-server-name>`)。

### 0x8024402F
**含义：** 外部 cab 文件处理完成，但出现一些错误。

**修复方法：** 这是一个客户端损坏问题。尝试手动修补机器。如果失败，您可能需要重置 Windows Update 组件（重命名 `SoftwareDistribution` 和 `catroot2` 文件夹）。



## 总结

在云中排查打补丁问题最终归结为标准的 Windows Server 管理。Azure Update Manager 充当编排器，但繁重的工作仍然由 Windows Update Agent 完成。每当您看到 `HRESULT` 错误时，获取代码，检查您的网络，确保有足够的磁盘空间，并验证手动打补丁是否有效。