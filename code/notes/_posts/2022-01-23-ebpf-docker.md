---
layout: post
title: 如何在MacOS/Win上构建能跑eBPF的Docker镜像
image: 
  path: /assets/img/blog/jeremy-bishop@0,5x.jpg
description: >
  不是简单就是了，有着时间不如使用 Vagrantfile
sitemap: false
comments: true

---

[Here is Github resp.](https://github.com/Jun10ng/ebpf-for-desktop)

## What
一个能在桌面系统上运行eBPF程序的Docker，主要用于学习eBPF.

Ubuntu: 21.04 

Kernel: 5.10.76

## Run it!

```
git clone https://github.com/Jun10ng/ebpf-for-desktop.git
cd ebpf-for-desktop/
docker build -t ebpf:v1 .
sh ./docker-run.sh
```

## How

由于一开始不知道`Mac`作为宿主机构建出来的`Linux`镜像是会缺少一些内核文件，走了许多弯路。

后续决定基于这个[专门给桌面系统构建内核的镜像](https://hub.docker.com/r/docker/for-desktop-kernel)做二次开发，
依旧会提示`Kconfig.h`文件缺少`generated/autoconf.h`, 根本原因是因为`linux-headers-$(uname -r)` 不存在，

那没办法，只能自行编译了。（以上两个脚本我集成到Dockerfile里了，不需要自己运行）


首先从国内镜像上下载对应版本的源文件：
see script [here](https://github.com/Jun10ng/ebpf-for-desktop/blob/main/linuxkit-dl.sh).

然后编译：
see script [here](https://github.com/Jun10ng/ebpf-for-desktop/blob/main/linuxkit-complier.sh)