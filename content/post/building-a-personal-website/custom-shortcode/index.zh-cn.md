+++  
author = "Lucas Huang"  
date = '2025-09-02T10:30:22+08:00'  
title = "为你的 Hugo 博客添加自定义短代码"  
categories = [  
    "Hugo Blog"  
]  
tags = [  
    "Shortcodes",
    "Buy Me a Coffee"  
]  
image = "cover.png"  
draft = false  
+++  
我将以 Buy Me a Coffee 按钮为例。  

## 为什么要在 Hugo 博客里添加 **Buy Me a Coffee**？

如果你分享知识、运营开源项目，或者只是单纯写博客，给读者一个方便的 “Buy me a coffee” 按钮是一种行之有效的募捐方式。在 Hugo 站点中，最简洁且易于维护的方法是把这段脚本封装进一个 **短代码（Shortcode）**。

## 步骤指南  
1. 从 [Buy Me a Coffee 官网](https://studio.buymeacoffee.com/) 拿到你的按钮代码。  
```
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="lucaslifes" data-color="#FFDD00" data-emoji="☕️"  data-font="Cookie" data-text="Buy me a coffee" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>
```

2. 在 `~/layouts/_shortcodes/` 下创建 `buymeacoffee.html`，把官方脚本完整粘贴进去。  

3. 在任意 Markdown 文件里调用该短代码：  
   ```markdown
   文章内容 …

   {{</* buymeacoffee */>}}

   更多内容 …
   ```

3. 如有需要，允许原始 HTML：  
   对于低于 0.116 的 Hugo 版本，在 `config.toml` 中启用 HTML：  
   ```toml
   [markup.goldmark.renderer]
   unsafe = true
   ```  
   Hugo 0.116 及以上版本已默认允许短代码内部的 HTML。  

## 优缺点  

- ✅ **可复用**：只需在一个地方修改 slug 或颜色。  
- ✅ **易维护**：脚本更新集中管理。  

## 效果预览  

{{< buymeacoffee >}}

---  

祝写博愉快，也感谢你的咖啡！☕️