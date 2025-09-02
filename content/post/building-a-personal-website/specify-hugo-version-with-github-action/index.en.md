+++  
author = "Lucas Huang"  
date = '2025-09-02T11:30:22+08:00'  
title = "Fixing Hugo Shortcode Errors During Azure Static Web Apps Deployment with Github Action"  
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
## Issue

I recently started using Hugo shortcodes. Everything compiles and runs fine locally:

```powershell
PS C:\Users\lucashuang\LucasLocalWorkspace\Blogs\lucaslifes> hugo --gc --minify --enableGitInfo
Start building sites …
hugo v0.146.4-985af1c0 windows/amd64

| EN  | ZH-CN
|----|----
| 139 | 72 ...

Total in 2683 ms
```

However, the build fails when deploying to Azure Static Web Apps via GitHub Actions:

```text
Start building sites …
hugo v0.124.1-db083b05 linux/amd64

Error: error building site: process: readAndProcessContent:
/github/workspace/content/page/about/index.md:21:1:
failed to extract shortcode: template for shortcode "buymeacoffee" not found
---End of Oryx build logs---
Oryx has failed to build the solution.
```

## Root Cause

Azure Static Web Apps uses **Oryx** as its default builder.  
The Hugo version bundled with Oryx is **v0.124.1**, but my custom `buymeacoffee` shortcode relies on features introduced in **≥ v0.146** (specifically [named arguments for `.Get`](https://github.com/gohugoio/hugo/releases/tag/v0.146.0)).  
Because the version is too old, the shortcode cannot be parsed and the build aborts.

## Solution: Tell Oryx to Use a Newer Hugo Version

Oryx looks at the environment variable **`HUGO_VERSION`** to decide which version to install.  
Add it to the existing “Build And Deploy” step in your GitHub Action:

```yaml
# .github/workflows/azure-static-web-apps.yml
- name: Build And Deploy
  uses: Azure/static-web-apps-deploy@v1
  env:
    HUGO_VERSION: 0.146.4   # ← Instruct Oryx to use this version
  with:
    azure_static_web_apps_api_token: ${{ secrets.AZURE_STATIC_WEB_APPS_API_TOKEN }}
    action: "upload"
    app_location: "/"
    api_location: ""
    app_build_command: "hugo --gc --minify --enableGitInfo"
    output_location: "public"
```

Leave the other steps unchanged. Push your code again and the build and deployment should succeed ✅.

## Takeaways

1. The Hugo version bundled with Oryx is outdated.  
2. If your custom shortcodes depend on newer features, pin the Hugo version.  
3. You can switch versions quickly via the `HUGO_VERSION` environment variable without altering your existing CI/CD pipeline.