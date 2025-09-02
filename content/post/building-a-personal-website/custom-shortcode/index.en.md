+++  
author = "Lucas Huang"  
date = '2025-09-02T10:30:22+08:00'  
title = "Add Custom Short Codes to Your Hugo Blog"  
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
I will take Buy Me a Coffee button as an example. 

## Why Add **Buy Me a Coffee** to a Hugo Blog?

If you share knowledge, run an open-source project, or simply blog for fun, giving readers a handy “Buy me a coffee” button is a proven way to increase donations. In a Hugo site, the cleanest and most maintainable approach is to wrap the script in a **Shortcode**.


## Step-by-Step Guide
1. Grab your button code from [Buy Me a Coffee Website](https://studio.buymeacoffee.com/).
```
<script type="text/javascript" src="https://cdnjs.buymeacoffee.com/1.0.0/button.prod.min.js" data-name="bmc-button" data-slug="lucaslifes" data-color="#FFDD00" data-emoji="☕️"  data-font="Cookie" data-text="Buy me a coffee" data-outline-color="#000000" data-font-color="#000000" data-coffee-color="#ffffff" ></script>
```


2. Create `buymeacoffee.html` under `~/layouts/_shortcodes/`. Paste the official script as it is.


3. Invoke the Shortcode in any Markdown file

   ```markdown
   Post content …

   {{</* buymeacoffee */>}}

   More content …
   ```

3. Allow raw HTML if needed
   For Hugo versions below 0.116, enable HTML in `config.toml`:

   ```toml
   [markup.goldmark.renderer]
   unsafe = true
   ```

   Hugo 0.116+ already allows HTML inside shortcodes by default.



## Pros & Cons

- ✅ **Reusable**: Change slug or colors in one place.  
- ✅ **Maintainable**: Script updates are centralized.  




## Live Preview

{{< buymeacoffee >}}

---

Happy blogging, and thanks for the coffee! ☕️
