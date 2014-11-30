---
layout: post
comments: true
date: 2014-11-29 23:04:41+00:00
title: Mac新手兼菜鸟程序员升级Yosemite，挖坑填坑
categories:  
- Mac

---
# 目录 
* [1. 升级](#update)
* [2. Java挂了](#java)
* [3. Homebrew跪了,泪奔](#homebrew)
* [4. 总结](#summary)

周围小伙伴一个个都升级到Yosemite了，搞得我也蠢蠢欲动，恰巧手机跟电脑用的同一个appid，要在手机上使用1password需要把电脑升级下，才能使用icloud同步密钥数据，想想以后应该会不少情况，所以，趁早升级。

作为Mac新手和菜鸟程序员，所有问题解决均借助Google和小伙伴，有问题还请轻拍。之所以在此强调程序员，是因为有些问题只有程序员才会遇到。
# <a id="update">  1.升级 </a>
用惯了Windows和Linux，升级Mac还有点不习惯，更多的事有点虚，虽然看到周围小伙伴依然好好正常的用着Yosemite，但毕竟头一次，而且网上也说了Homebrew会带来些问题，连Mac都还没用熟就升级系统，还是有点怕。

首先，我没有对系统进行备份，这都是被周围人忽悠的，虽然Mac升级一般不会有问题，但还是要养成备份的好习惯，行走江湖，还是得防着点，万一跪了还能回来。而且Mac提供了Time Machine这种神器，方便快捷，不会用Time Machine？[点这里][1]

可以采用在线升级的方法，[官网][2]有教程。为了节省下载时间，我找某大师拷贝了个安装镜像，然后一路继续，经过多次重启电脑之后，终于进去了，界面不错。

![桌面截图](http://lxweiblog.qiniudn.com/2014-Yosemite_screen.png)

高兴劲儿还没过，就发现问题来了。可是没有备份，回不去了，只能硬着头皮往下走。

# <a id="java">  2.Java挂了</a>

Java挂了，所有依赖Java的开发工具都不能用了，看了池大大的[Blog][3]好像有写，赶紧去[下载][4]、安装，哈哈，终于又能用了。

# <a id="homebrew">  3.Homebrew跪了,泪奔</a>
Homebrew在我初始配置开发环境时提供了巨大的便利，但是，升级完却不能用了，看了下原因应该是Ruby版本变成2.0了，但既然有问题就慢慢解决了，还好是周末，慢慢折腾吧……

## 3.1 nginx
因为一直用nginx做服务器，所以，想先看下环境是否能跑起来，运行命令
>sudo nginx

报错：
>nginx: [emerg] mkdir() "/usr/local/var/run/nginx/client_body_temp" failed (2: No such file or directory)

上网搜索了下，很高兴的发现有人遇到了相同的问题([点击查看][5])，但是，考虑到nginx在Mavericks上运行正常，现在不对了，那么应该不只是因为没有这个文件，所以，我没有直接```mkdir```这个文件，而是采用了```brew update ; brew upgrade nginx```，结果发现并没有解决问题，而且，我也没打算再mkdir那个文件夹来试试，现在想想好像有点愚蠢，毕竟建立没有的文件夹才是收获最多赞的答案。

然后，我搜到了池大大的[blog][3]，谁让池大大在我心中是大神一般的存在呢，所以，不管三七二十一，直接执行了池大大给的解决方案。执行到```brew upgrade```时，报错了，nginx、mysql、redis、vim报错，其中mysql报错：
>C compiler not found, but GCC is installed

上网一搜，升级xcode时，把 **Command Line Tools** 干掉了，所以，运行命令：
>xcode-select --install

然后，要做的就是等待了，这个有点慢。

当安装完成后，重启电脑，然后运行```brew upgrade```，nginx、mysql、redis、vim全部升级成功，长舒一口气，爽！

然后，运行```sudo nginx```，采用ps命令查看进程，发现nginx正常运行，哈哈~

## 3.2 PHP没了
Mac自带了PHP，但是，为了后面统一采用Homebrew对开发环境进行管理，我自己安装了PHP，其中系统自带的PHP时5.4.24，自己安装的是5.4.32，通过修改**.profile**文件里的PATH环境变量，使系统默认使用5.4.32，然而，我现在运行命令：
>php -v

查看到php版本是5.5.19，然后运行命令：
>whereis php

发现是/usr/bin/php，然后使用locate命令查看，发现只有系统自带的php了，**通过Homebrew自行安装的php没了，没了**！

没有了再装就是了，所以，我很快使用```brew install php54```重装了php5.4，然后，居然还是找不到，但是，我在```/usr/local/Cellar/php54/```文件夹下找到了5.4.32和5.4.35两个版本，如下图所示：

![php-fpm](http://lxweiblog.qiniudn.com/2014-php-fpm.jpg)

然后，我运行
>php -v

显示版本是*5.4.35*，可是我运行```sudo php-fpm```，然后写了个页面输出 phpinfo() 却看到的是*5.4.32*，晕了。这里为什么是这样，有高手看到，还请告知，不胜感激。

然后，我到5.4.32目录下直接运行php-fpm:

!["/usr/local/opt/icu4c/lib/libicui18n.53.dylib" doesn't exist](http://lxweiblog.qiniudn.com/2014-libicui18n.53.dylib.jpg)

居然报错，放Google，搜到[答案][6]，按照提示操作，如上图所示，再次，启动，Done。哈哈，一切正常了。

# <a id="summary">  4.总结</a>
折腾了大半天，系统暂时好了，那么，问题来了：

1. 升级前是否该备份？虽然周围小伙伴都不备份，但我觉得还是备份吧，不然，各种硬盘里装那么多片也不能当干粮，何不给备份让点空间？
2. 在没有完全搞清楚原因的情况下，为什么要照抄池大大的解决方案？实际上，据其他小伙伴说，升级完系统，直接```brew update, brew upgrade```就搞定了，压根儿不用折腾。哎，可我为什么会倒腾出这事儿呢？还不是因为池大大是Mac界的偶像级人物，自己认为照抄就可以了，但实际上，池大大升级时，Yosemite都还没正式放出来，那会儿出的问题都不是我们这些屁民会遇到的，一个半月时间，苹果早修复了，不然，人家还能是苹果？还得变化的看待问题，Too young, too simple.
3. 菜鸟，活该。自己以前都用集成环境搞定了，甚至重来没用过nginx，刚开始折腾，对nginx + php + mysql 的架构还不熟悉，自己配置环境的能力还太弱，这方面得多研究研究，即使这次把环境搞定了，但还是有很多问题没搞清楚，努力吧，少年！



[1]:http://support.apple.com/zh-cn/ht1427
[2]:https://www.apple.com/hk/osx/how-to-upgrade/
[3]:http://weibo.com/p/1001603766056865935197
[4]:http://support.apple.com/kb/DL1572?viewlocale=en_US&locale=en_US
[5]:http://stackoverflow.com/questions/26450085/nginx-broken-after-upgrade-to-osx-yosemite
[6]:https://github.com/Homebrew/homebrew-php/issues/1421