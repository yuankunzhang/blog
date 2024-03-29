+++
title = "TTY、终端和控制台：历史、用途和区别"
date = 2022-05-30T21:20:45+08:00
slug = "tty-terminal-console"
categories = ["linux", "terminal"]
tags = ["linux", "tty", "terminal", "console"]
draft = false
+++

在计算机的世界里，许多术语和概念是从早期的计算机技术中沿袭而来的，包括 TTY、终端和控制台。了解这三个术语的历史、用途以及区别对理解操作系统的终端和伪终端子系统很有帮助。

TTY（Teletype，中文一般翻译成电传打字机）的历史比计算机的历史更久远，可以追溯到电报机的时代。在电脑还没出现之前，人们通过这种设备进行远程信息传输[^1]。随着时间的推进，TTY 在计算机领域中的含义演变成为一种能够在用户与计算机系统之间通过文本形式进行交互的接口。

<!-- more -->

[^1]: [Royal Earl House Invents a Telegraph that Composes and Prints in Roman Characters Rather than in Code](https://www.historyofinformation.com/detail.php?id=5483)

![](/img/printing-telegraph.jpg)
(Hughes Telegraph，图片来源：Wikipedia)

![](/img/siemens-t37h-without-cover.jpg)
(Siemens t37h，图片来源：Wikipedia)

终端（Terminal）最初是指通过线路连接的物理设备，用户可以通过键盘输入并从屏幕读取信息。在大型机时代，计算机不仅昂贵，还显得庞大而笨重。计算机的使用者则借助“终端设备”来连接到主机，并以命令行的方式进行交互操作。最早被用作终端设备的是电传打字机与打字机的组合，电传打字机负责输入，而打印机则将输出结果打印在纸张上。不久之后，衍生出了专门的终端设备，将电传打字机和打印机的功能合二为一。

![](/img/the-ibm-2741-terminal.jpg)
([The IBM 2741 Terminal](http://www.columbia.edu/cu/computinghistory/2741.html)，图片来源：[IBM](http://www-03.ibm.com/ibm/history/ibm100/images/icp/Z491903Y91074L07/us__en_us__ibm100__selectric__selectric_2__900x746.jpg))

若干年后，电子显示屏逐渐取代了打印机成了新的输出设备。我们所熟知的”物理终端“的概念指的就是这一时期的终端设备，其中比较有代表性的是德州仪器的 VT100 系列终端设备。

![](/img/dec-vt100-terminal.jpg)
(DEC VT100 terminal，图片来源：Wikipedia；[这个网站](https://www.vt100.net/)包罗了关于该系列终端设备的详尽资料)

随着时间的推移，我们进入到了计算机时代。小型机的性能已经足够强大，使得人们无需再依赖专门的终端设备来分时使用大型机。虽然专用的终端设备被时代淘汰掉，但终端作为“使用文本进行人机交互的用户界面”的概念却延续了下来。在这个时代中，个人计算机不仅自身作为主机，同时也肩负起了终端设备的角色。

![](/img/att-unix-pc-7300.jpg)

(AT&T UNIX PC 7300，图片来源：[oldcomputers.net](https://oldcomputers.net/att-unix-pc.html))

随着计算机技术的发展，终端的概念也开始扩展到软件，形成了模拟终端（Terminal Emulator），它在图形用户界面环境中模拟了物理终端的功能。我们今天广泛使用的 xterm，gnome-terminal 等软件，都属于模拟终端。

控制台（Console）是操作系统层面的抽象概念，它指的是用户用来控制操作系统行为的主接口。虽然控制台和终端这两个术语经常被互相替换使用，但它们的侧重点并不相同。更加令人困惑的是，在不同的场合，我们经常会看到各种不同的名称来指代控制台，比如终端、虚拟终端、虚拟控制台等。

我们可以把控制台理解为操作系统通向外部的桥梁：用户通过控制台向操作系统下达命令，而操作系统则会将命令的执行状态和结果反馈到控制台进行显示。在操作系统的另一边，也就是用户端，这个桥梁与一个终端设备相连，这个终端设备可以是物理设备，也可以是虚拟设备。又因为最早使用的终端设备是电传打字机，所以在 Unix 系统中的终端子系统又被称为 TTY 子系统。

以上就是 TTY、终端和控制台这三个术语的历史渊源及其概念性的差异。在后续的一篇文章中，我将深入探讨这三个术语在 Linux 语境下的具体含义和应用。
