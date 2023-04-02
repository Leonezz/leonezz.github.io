---
title: Ubuntu(WSL2) 编译安装Qt-5.15.2
date: 2021-08-02 15:24:43
update: 2021-08-02 15:24:43
tags:
    - Qt
categories: 记录备忘
---

以往都是通过离线安装包或者在线安装器安装Qt，自动Qt不提供离线安装包以后，在某些无法联网的场景下只能通过源码编译安装。本次在 WSL 的环境下虽然可以联网，但权当一次体验编译安装的机会。

<!--more-->

首先从[qt的IO网站](https://download.qt.io/official_releases/qt/5.15/)或者[清华镜像站](https://mirrors.tuna.tsinghua.edu.cn/qt/archive/qt/)或者其他任何托管有qt源码的镜像站下载源代码（以5.15.2为例）。然后需要准备编译环境，这里建议参考Qt的[编译指导](https://wiki.qt.io/Building_Qt_5_from_Git)来安装编译所需要的环境。

安装完成后，就可以开始配置构建选项。这里建议在源代码的平行目录中进行，即假设源代码的目录是：`~/qt-src`，那么建议在类似 `~/qt-build` 的目录中进行。在构建目录中，运行 `./configure` 脚本可以进行编译前的配置，`configure` 脚本有很多可选的参数，具体的列表可以打开源码目录下的 `qtbase/config_help.txt` 查看。

`configure` 脚本执行完毕后，如果脚本的输出没有提示环境的不支持，就可以执行 `make` 开始编译。这里因该需要编译一天左右，我的机器上就从下午三点半编译到第二天的十二点四十，大约21小时。

编译完成后就可以执行 `make install` 进行安装，默认的安装目录是 `/usr/local/Qt-5.15.2`。此时执行 `qmake` 是找不到目标的，需要手动添加到 `PATH` 或者使用 `qtchooser` 工具。以 `qtchooser` 为例，它所默认的qt路径应该是 `/usr/lib/x86_64-linux-gnu/qt5`。因此需要更改它的配置文件，打开文件 `/usr/lib/x86_64-linux-gnu/qt-default/qtchooser/default.conf`，将内容改成：

```conf
/usr/local/Qt-5.15.2/bin
/usr/local/Qt-5.15.2/lib
```

就完成了。