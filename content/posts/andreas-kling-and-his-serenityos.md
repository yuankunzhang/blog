+++
title = "传奇程序员 Andreas Kling 和他的 SerenityOS"
date = 2022-09-14T23:18:00+08:00
categories = ["misc"]
draft = false
+++

我们今天故事的主角，是一个叫做[Andreas Kling](https://awesomekling.github.io/about/)的瑞典程序员。

这个月的 9 月 12 日，Andreas 在他的个人网站上刊出了[一篇文章来介绍他的Ladybird浏览器项目](https://awesomekling.github.io/Ladybird-a-new-cross-platform-browser-project/)。浏览器大概是这个星球上最为庞大的软件项目，从零开始写出一个即便只是玩票性质的浏览器也是一项惊人困难的任务。如果你对它的难度没有直观认识的话，不妨猜一猜 Firefox 浏览器总共有多少行代码？答案是[2100万行](https://hacks.mozilla.org/2020/04/code-quality-tools-at-mozilla/)！[数量庞杂（并且仍在爆炸式增加）的各种web标准](https://www.w3.org/TR/)使得编写浏览器几乎成了只有互联网寡头才能组织起人力和资源来完成的事情。所以，Andreas 的这个几乎以一己之力做出来的浏览器完全值得让人拍手称奇。

{{< figure src="https://awesomekling.github.io/assets/ladybird-acid3.png" caption="<span class=\"figure-number\">Figure 1: </span>Ladybird 浏览器以满分的成绩通过了[Acid3](https://en.wikipedia.org/wiki/Acid3)测试" >}}

事实上，在 Ladybird 项目之前，Andreas 已经完成了多项壮举，比如一个叫做[SerenityOS](https://github.com/SerenityOS/serenity)的操作系统和一门叫做[Jakt](https://github.com/SerenityOS/jakt)的系统级编程语言。用他自己的话说，他的目标是“从头编写一个完整的操作系统”。这些项目，不论单独拎出来哪一个，都显得过于庞大。如果这些项目的代码不是实实在在摆在大家的面前的话，我一定觉得这个人是痴人说梦。

而所有这些项目的起点，都可以追溯到 Andreas 下定决心戒除毒瘾的那个秋天。

没错，Andreas 曾经沾染毒瘾，是一名瘾君子。2018 年 10 月份，在结束了 3 个月的戒断治疗后，为了打发漫长难遣的时间，他开始疯狂地写起了代码。一开始，他完成了一个可执行文件的解析器。渐渐地，他又陆续写出来一个文件系统浏览器和一个图形界面框架。这时候 Andreas 惊奇地发现，一个简易的操作系统其实已经呼之欲出了。于是他将这若干个基础部件组合成一个操作系统，并称之为 SerenityOS。

{{< figure src="https://raw.githubusercontent.com/SerenityOS/serenity/master/Meta/Screenshots/screenshot-b36968c.png" caption="<span class=\"figure-number\">Figure 2: </span>这是一款吸收了 90 年代美学理念的类 Unix 系统" >}}

这个由 Andreas 在 2018 年单枪匹马创建的项目，到现在已经蔚为大观。不同于浏览器，从头开始写出一个简单的操作系统并非难事，难的是聚拢各路牛人，形成真正有活力的社区。截至今日，该项目已经斩获了两万一千多颗 Github 星标，共有超过 700 位开发者向该项目贡献了代码。在我看来，Andreas 的这个操作系统谈不上有什么实际的用途，最多只是一个稍具规模的玩具而已。不过，不少重量级的软件都是“玩”出来的，难道不是吗？比如 Linux，这个在今天已经无处不在的操作系统（你甚至[在火星上也能发现它的存在](https://www.pcmag.com/news/linux-is-now-on-mars-thanks-to-nasas-perseverance-rover#:~:text=The%20helicopter%2Dlike%20drone%20on,operating%20system%2C%E2%80%9D%20Canham%20said.)），Linux Torvalds 最初发布它的时候可没有想到有一天它竟会大放异采。

{{< figure src="https://i.pcmag.com/imagery/articles/00lMJJf4MDevP6tyVC7fegz-1.fit_lim.size_1600x900.v1613762576.jpg" caption="<span class=\"figure-number\">Figure 3: </span>图片来源：NASA" >}}

时至今日，Andreas 依然耕耘不辍，几乎全年无休。我们就以他的 Github 状态墙的截图结束本文吧，希望他能在写代码的道路上继续快乐地走下去。

{{< figure src="/ox-hugo/andreas-kling-github.png" >}}
