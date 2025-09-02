+++  
author = "Lucas Huang"  
date = '2025-09-02T11:30:22+08:00'  
title = "修复Hugo使用Github Action部署时遇到的找不到短代码报错"  
categories = [  
    "Hugo Blog"  
]  
tags = [  
    "Shortcodes",
    "Azure Static Web App",
    "Hugo"
]  
image = "cover.png"  
draft = false  
+++  
## 问题现象  
最近在使用 Hugo 的短代码功能，在本地调试并且构建完全正常  
```powershell
PS C:\Users\lucashuang\LucasLocalWorkspace\Blogs\lucaslifes> hugo --gc --minify --enableGitInfo  
Start building sites …  
hugo v0.146.4-985af1c0 windows/amd64  

| EN  | ZH-CN  
|----|----  
| 139 | 72 ...  

Total in 2683 ms  
```  

但是使用Github Action部署到Azure Static Web Apps则报错。
```text
Start building sites …  
hugo v0.124.1-db083b05 linux/amd64  

Error: error building site: process: readAndProcessContent:  
"/github/workspace/content/page/about/index.md:21:1":  
failed to extract shortcode: template for shortcode "buymeacoffee" not found
---End of Oryx build logs---
Oryx has failed to build the solution.
```  

## 原因分析  
Azure Static Web Apps 默认使用 **Oryx** 构建器。  
Oryx 内置的 Hugo 版本只有 **v0.124.1**，而我自定义的 `buymeacoffee` shortcode 依赖 **≥ v0.146** 的新特性（即 [`.Get` 支持命名参数](https://github.com/gohugoio/hugo/releases/tag/v0.146.0)）。  
版本过低 → 解析 shortcode 失败 → 构建直接中断。  

## 解决方案：告诉 Oryx 使用新版本 Hugo  
Oryx 通过环境变量 **`HUGO_VERSION`** 识别所需版本。  
在现有 GitHub Action 中的 “Build And Deploy” 步骤加入即可：  

```yaml
# .github/workflows/azure-static-web-apps.yml
- name: Build And Deploy
  uses: Azure/static-web-apps-deploy@v1
  env:
    HUGO_VERSION: 0.146.4   # ← 告诉 Oryx 使用此版本
  with:
    azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}
    action: "upload"
    app_location: "/"              
    api_location: ""
    app_build_command: "hugo --gc --minify --enableGitInfo"
    output_location: "public"
```

其他步骤不变。再次推送代码后，构建与部署顺利通过 ✅。  

## 小结  
1. Oryx 默认 Hugo 版本偏旧。  
2. 自定义 shortcode 如果用到新特性，务必锁定版本。  
3. 通过环境变量 `HUGO_VERSION` 可快速切换，不影响现有 CI/CD 流程。  
