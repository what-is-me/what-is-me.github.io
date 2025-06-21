---
mathjax: true
title: "Centos7 使用指北"
date: 2025-06-21 19:45:28
tags:
    - "CentOS7"
category: "系统配置"
---
有的时候我们不得不在古老的，稳定的，“主流的”CentOS7系统上开发软件，但是该系统上不少软件版本较为落后。特别在没有root权限来使用yum安装软件时。

网上的大多数教程，比如"在CentOS7上安装XXX"都是下载源代码编译安装。这样需要手动管理依赖，非常麻烦。

我们可以使用[Reference](https://stackoverflow.com/questions/36651091/how-to-install-packages-in-linux-centos-without-root-user-with-automatic-depen) 的方法：

安装Miniconda3：

```bash
curl https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh > Miniconda.sh
bash Miniconda.sh -p ~/conda
```

之后就可以用conda的包管理器了，比如安装gcc，clang等，比如：

```bash
conda install -c conda-forge gcc=13.2.0
```

注意要在更新gcc/clang设置环境变量：

```bash
export LD_PRELOAD=~/conda/lib/libstdc++.so.6
```
