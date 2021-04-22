---
title: "git忽略已提交过的文件"
date: 2021-04-14T14:55:38+08:00
draft: false
tags: ["git"]
---

.gitignore可以把没有提交过的文件忽略，那如果是已经提交的文件，本地修改了但是又不想提交，也不想在git status的时候看到，这个时候下面的指令就可以做到

```bash
git update-index --assume-unchanged [file-path]
```

如果之后又想继续版控这些文件，下面的指令可以恢复

```bash
git update-index --no-assume-unchanged [file-path]
```

如果忘记了哪些文件被屏蔽了，可以用下面的指令查看

```bash
git checkout-index -a
```
