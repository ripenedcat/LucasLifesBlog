+++ 
author = "Lucas Huang"  
date = '2025-08-31T10:00:00+08:00'  
title = "Delete Stubborn nul File on Windows"  
categories = [
    "Other Tech"
]  
tags = [
    "nul file",
    "Git Bash",
    "Windows Tips"
]  
image = "cover.png"  
# draft = false  
+++


## Background

While running a script on Windows 10 I accidentally created a file literally called **`nul`** (no extension).  
Because **nul** is a reserved device name in DOS/Windows, File Explorer and the classical `del` command refuse to remove it.

## Popular but Ineffective Work-arounds

Search results usually recommend something like:

```bat
rd /s \\.\C:\aux 
del \\.\C:\temp\nul.exe
```

Those commands leverage the Win32 “device namespace” (`\\.\`) to bypass the restriction.  
Unfortunately they didn’t work for me—probably due to path, ACL or modern Windows internals.

## The Working Solution: Git Bash

A comment on Stack Overflow saved the day. If you have **Git for Windows** installed:

1. Navigate to the directory that contains the troublesome **`nul`** file;  
2. Right-click an empty area and choose **“Git Bash Here”**;  
3. Run:

   ```bash
   rm nul
   ```

   The file disappears instantly.

### Why does it work?

`rm` shipped with Git Bash comes from MSYS2/MinGW and follows POSIX semantics,  
so it treats **`nul`** as an ordinary filename instead of a special device, and performs a normal unlink syscall.

## Takeaways

* Windows reserved names (con, prn, aux, nul, …) are legacy traps;  
* When stuck, try deleting them from a POSIX-like environment such as Git Bash, MSYS2 or WSL;  
* Better yet, never generate those names in scripts or code.

## References
- [Stack Overflow – "Delete a file named ‘NUL’ on Windows"](https://stackoverflow.com/questions/17883481/delete-a-file-named-nul-on-windows)
