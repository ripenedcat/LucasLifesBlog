+++
author = "Lucas Huang"
date = '2025-12-29T10:00:00+08:00'
title = "如何移除 Azure Foundry/OpenAI 模型的内容过滤器"

categories = [
    "Azure"
]
tags = [
    "Azure Foundry",
    "Azure OpenAI"
]
image = "cover.png"
# draft = true
+++

## 背景

在使用 Azure Foundry AI/ Azure OpenAI 部署模型时，系统默认会应用 `Microsoft.DefaultV2` 内容过滤器。虽然内容过滤器的初衷是为了确保生成内容的安全性和合规性，但在某些场景下，过于严格的过滤可能会导致以下问题：

- 模型回答被截断或不完整
- 输出内容不通顺，影响可读性
- 正常的技术讨论被误判为敏感内容

对于开发测试环境或特定业务场景，我们可能需要移除或调整这个内容过滤器。

## 解决方法

通过以下步骤，我们可以在创建模型部署时将内容过滤器从 `Microsoft.DefaultV2` 修改为 `Microsoft.Nil`（无过滤）。

### 打开浏览器开发者工具

在 Azure Foundry Portal中，准备创建新的模型部署前或者修改已有模型前，按 `F12` 打开浏览器的开发者工具（Developer Tools）， 并切换到网络标签页。下图时我以创建 `DeepSeek-V3.2` 为例

![打开开发者工具捕获网络请求](network-capture.png)

### 创建/修改模型部署并捕获请求

在 Azure Foundry Portal 中点击提交，我们会看到 `DeepSeek-V3.2` 被创建，且内容过滤器为 `DefaultV2`。 

此时开发者工具已经捕获到提交给后台的请求。在网络请求列表中，找到创建部署的 API 请求（第一个 POST 请求到 `management.azure.com` 端点）。

右键点击该请求，选择 **Copy（复制）** → **Copy as PowerShell（复制为 PowerShell）** 。

![复制请求为PowerShell指令](copy-as-powershell.png)


### 修改请求参数

打开文本编辑器， 这里以Windows自带的 **PowerShell ISE** 为例，粘贴刚才复制的命令。

在最后一行的请求的 JSON payload 中找到 `raiPolicyName` 字段，将其值从 `Microsoft.DefaultV2` 改为 `Microsoft.Nil`

![去除内容过滤器](remove-content-filter.png)

### 执行修改后的请求

在 PowerShell ISE 中执行修改后的命令，我们可以看到一个200 OK的返回结果。

![修改请求成功](request-succeed.png)

这时回到Azure Foundry Portal, 可以看到内容过滤器已被移除。

![内容过滤器已被移除](removed-content-filter.png)


## 总结

通过捕获和修改 Azure Portal 的 API 请求，我们可以在创建模型部署时指定 `Microsoft.Nil` 内容过滤器，从而移除默认的内容限制。这种方法适用于需要更灵活输出控制的开发测试场景，但请务必评估安全和合规风险，在生产环境中谨慎使用此方法。
