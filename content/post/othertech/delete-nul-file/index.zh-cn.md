+++ 
author = "Lucas Huang"  
date = '2025-08-31T10:00:00+08:00'  
title = "在 Windows 上删除顽固的 nul 文件"  
categories = [
    "Other Tech"
]  
tags = [
    "nul file",
    "Git Bash"
]  
image = "cover.png"  
# draft = false  
+++


## 背景

某次在 Win10 上使用脚本时意外生成了一个名为 **`nul`** 的文件（注意：不是扩展名为 `.nul` 的文件，而是文件名就叫 **nul**）。  
这个文件几乎无法用资源管理器或 `del` 命令删除，因为 **nul** 是 DOS 时代的保留名，代表“黑洞设备”。

## 常见却无效的做法

互联网上最流行的方案大概是下面两条命令：

```bat
rd /s \\.\C:\aux 
del \\.\C:\temp\nul.exe
```

它们利用 Win32 命名空间绕过保留字，但在我的机器上尝试后仍然失败。  
原因可能是路径、权限或 Win10 的实现细节不同，简单粗暴的 **`rd` / `del`** 并不保险。

## 最终成功的解决办法：Git Bash

在 Stack Overflow 上翻到一条评论，亲测有效，步骤如下（前提是安装过 Git for Windows）：

1. 打开包含 **`nul`** 文件的目录；  
2. 在空白处右击，选择 **Git Bash Here**；  
3. 直接输入

   ```bash
   rm nul
   ```

   干净利落地解决。

### 为什么 Git Bash 能行？

Git Bash 内置的 `rm` 来自 MSYS2/MinGW，底层使用 POSIX 接口，不会去解析 DOS 保留名；  
对它来说 **`nul`** 只是普通文件名，因此能够正常 unlink。

## 小结

* Windows 下的保留设备名（con、prn、aux、nul 等）让很多历史兼容问题变得棘手；  
* 若遇到无法删除的保留名文件，可优先尝试 **Git Bash / MSYS2 / WSL** 等类 Unix 环境；  
* 最保险的还是避免在脚本或代码里生成这些名字。

## 5 参考资料

- [Stack Overflow – "Delete a file named ‘NUL’ on Windows"](https://stackoverflow.com/questions/17883481/delete-a-file-named-nul-on-windows)