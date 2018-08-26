---
layout: post
title: "[#snippet] How to reach and browse Windows paths rapidly using Bash on Windows
  (a.k.a. Windows Subsystem For Linux)"
date: '2018-02-09 09:35:53'
tags:
- snippet
- wsl
- bash-on-windows
- cdw
---

Hi,

Here is a simple snippet I've put in my WSL `.bashrc` file to rapidly reach Windows paths when on Bash for Windows.

```
cdw() {
    local winpath="$1"
    winpath=$(echo $winpath | sed -e 's/\\$//g' -e 's/C\:/\/mnt\/c/g' -e 's/\\/\//g')
    cd "$winpath"
}
```

Place it in your .bashrc, then you can type `cdw 'C:\What\Ever\Path'` and you will reach `/mnt/c/What/Ever/Path`. 

Hint:
You can use the command `echo $winpath | sed -e 's/\\$//g' -e 's/C\:/\/mnt\/c/g' -e 's/\\/\//g'` alone to translate the Windows path into the corresponding WSL path in your scripts.