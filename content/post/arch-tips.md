---
title: "ArchLinux Tips"
date: 2022-07-02T08:12:37+08:00
draft: false
---

## AUR (Arch User Repository) 使用

参考 [AUR](https://wiki.archlinux.org/title/Arch_User_Repository)
* 使用 git 拉取目标仓库: `git clone xxx.git`;
* cd 到相应目录, 执行 `makepkg`;
* 执行 `pacman -U xxx.zst` 安装;
* 可以使用 `git clean -dfx` 清除历史文件;
