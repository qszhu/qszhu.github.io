---
layout: post
title: 写在 Apple Vision Pro “停产”之前
date: 2025-01-19 00:00:00 +0800
tags: [apple, vision_pro]
---

### 目录
* 《[Apple Vision Pro 初体验](/2024/02/17/apple-vision-pro-first-impressions.html)》
* 《写在 Apple Vision Pro “停产”之前》 <-- 你在这里

距离[上一篇](/2024/02/17/apple-vision-pro-first-impressions.html)已经快一年了, 这意味着Apple Vision Pro发售已经一年. 然而没想到却迎来了“停产”的消息[1]. 唱衰VR/AR的人可能觉得自己的判断再次得到验证, 然而我却不以为然. 自从购入了AVP后的这半年时间, 我几乎每天都会使用超过数小时, 并且已经回不去了. 这里就总结下半年来深度使用AVP的感受.

### 最大亮点

个人以为AVP的最大亮点还是提供无限屏幕摆放空间的能力. 传统的VR设备在一定程度上也可以实现这样的功能, 比如提供一个巨大荧幕的虚拟影院. 但AVP在此基础上有两个非常重要的改进. 一是显示质量, 几乎看不到纱窗效应; 二是与真实环境的融合, 戴着AVP的时候喝个水签个字刷个手机完全没有障碍. 结合AVP的便携性, 就给移动办公的体验带来了质的飞跃. 众所周知, 一些职业非常需要大量的显示空间, 比如程序员/设计师/交易员/职业玩家等, 经常可以看到他们同时使用多个显示器. 而使用AVP的话就可以随时随地复刻这种多显示器的体验.

除了能在移动的场合复刻多显示器的体验之外, 另一个可能容易被忽略的优点便是私密性. 我在公司的工位头顶上就是一个能看到公司全貌的监控摄像头, 显示器上的内容自然也完全暴露在监控下. 而使用AVP的话除了自己就没人能看到你的屏幕, 缺点则是没法指着你屏幕上的内容给边上的人进行讲解.

### 移动办公

因为工作关系, 我对于移动办公一直有刚需. 之前也探索过不同的方案, 最后都是基于远程桌面构建的, 比如Raspberry Pi或是[SteamDeck](/2023/12/03/steamdeck.html). 这类方案需要以下组件:

* 远程桌面

AVP上有原生的Screens(收费)[2], 以及官方的虚拟显示器, 都可以满足基本的日常需求. 在系统更新到最新版本之后, 虚拟显示器还升级成了曲面屏, 相比同等大小的物理曲面屏就具备了成本优势[3].

![monitor](/assets/images/2025-01-19/monitor.jpg)

*Image credit: Apple*

* VPN

官方的虚拟显示器只能连接到局域网中的Mac, 要连接局域网外的主机就需要VPN. Screens可以使用Tailscale. 我个人一直使用的方案则是Wireguard, 通过一系列软件包将Raspberry Pi作为设备接入VPN的透明代理, 因此并没有直接在AVP上安装Wireguard. 有机会可以展开说说.

* 键鼠

有一说一, AVP的虚拟键盘属于一眼看上去非常惊艳, 实际用起来效率很低的玩意儿. 好在可以使用蓝牙键盘. 据说用苹果的Magic Keyboard的话会被自动识别和跟踪, 实现输入提示框跟随键盘移动, 或是在沉浸模式下也能看到键盘. 然而在尝试连接我们家iMac的键盘(需换电池的版本)时却提示设备不支持, 可能是键盘太老旧的关系. 虽然第三方的蓝牙键盘也可以用, 不过就没有上述好处了.

第三方的蓝牙鼠标一开始也是不完全支持的, 只支持Magic Trackpad, 体现了苹果一贯的小气. 好在系统更新到最新版本后, 第三方的蓝牙鼠标也都可以用了.

这样看下来将AVP用于基于远程桌面的移动办公的话, 就只有显示效果是革命性的, 其他方面则都不是开箱即用的, 谈不上完美. 那么如果都用原生的App呢?

### 原生App

这时就暴露出AVP上软件的匮乏了. 虽然AVP可以兼容iPad上的App, 但为iPad设计的App不一定能直接在AVP使用. 最常见的例子便是微信. iPad版微信登录需要手机扫码. 现在给你三秒钟思考一下: 手机要怎么扫到贴脸显示的二维码? 三, 二, 一, 时间到!

我最后使用的方法是将AVP的画面镜像到外部显示器. 比如在Raspberry Pi上跑一个UxPlay[4]就能让普通的显示器支持AirPlay镜像. 然而说起来简单, 实际操作起来仍旧很别扭, 因为镜像出来的画面是会跟着你的头部移动的, 需要预判好当你盯着二维码看的时候二维码会出现的位置(比如屏幕中心), 先用手机对着那里, 再看向二维码(根据经验视线必须集中在二维码上, 因为AVP视野中心以外的画面质量会降级, 造成扫码失败), 别扭的程度谁用谁知道.

### 杀手应用

可以说AVP革新了我的工作方式, 因此我现在已经回不去了. 那么AVP对于非专业用户的吸引力在哪里呢? 比较遗憾的是AVP发售一年后原生的应用依旧不多, 对于国区来说更是如此.

在[上一篇](/2024/02/17/apple-vision-pro-first-impressions.html)中我提到AVP的杀手应用之一是看3D电影. 目前腾讯视频的SVIP会员可以看到寥寥几部腾讯参与发行的3D电影, 比如<毒液>等. 说起来AVP版本的腾讯视频是做了手机登录的, 所以微信不做AVP版本不是不能而只是不想而已吧. 这可能也体现了当前互联网大厂对于AVP的态度: 现有的产品形态并不需要AVP提供的加成, 现有的iPad版本既不需要额外投入又可以收获新用户, 就没必要投入开发新的AVP版本.

相比之下一些小开发商倒是在探索能发挥AVP潜力的新产品. 举几个国区可以下载到的例子:

* Art Projector: Da Vinci Eye[5]: 可以把照片投射到平面上, 作为手绘时的参照;
* Simply Piano[6]: 学习钢琴的app, 应该还是一个技术演示, 内容比较少, 但已经充分利用到了AVP的空间计算能力.

### 小结

个人感觉造成AVP渗透率不高的现状可能还是因为苹果的投入不足. 比如从使用上来讲, 对于键鼠的支持一开始就是缺失的. 又比如从开发上来说, 苹果提供的接口也有各种限制. 举个最简单的例子, 任何VR设备都会在发售时自带类似“空间画布”的app, 可以在虚拟空间中操作笔刷画画. 而AVP中要等到设备发布了将近一年, 系统升级了一个大版本, 增加了一些新的接口之后, 才有对应的官方Demo. 再举个例子, 在官方的虚拟曲面显示器发布之前, 我原本准备自己实现一个的, 但发现用高层的接口(SwiftUI)实现不了, 用底层的接口(Metal)又异常繁琐和受限. 官方实现的功能从预告到发布也经历了大半年时间, 应该很能说明问题了.

将AVP与初代iPhone对比的话, 其实AVP的发展速度并不慢, 至少第一时间就有了App Store和大量兼容App. 但初代iPhone似乎并没有整出发售一年就停产的活. 作为果粉来说, 对于AVP确实有些怒其不争了.

### 参考

* [1] [Apple Vision Pro May Now Be Out of Production](https://www.macrumors.com/2024/12/31/vision-pro-may-be-out-of-production/)
* [2] [Screens 5: VNC Remote Desktop](https://apps.apple.com/us/app/screens-5-vnc-remote-desktop/id1663047912?platform=vision)
* [3] [Apple Vision Pro's killer feature is finally here - and made my $3,500 investment more worth it](https://www.zdnet.com/article/apple-vision-pro-finally-supports-ultra-wide-displays-but-is-it-worth-the-price-here-are-the-results/)
* [4] [Github - FDH2/UxPlay: AirPlay Unix mirroring server](https://github.com/FDH2/UxPlay)
* [5] [Art Projector: Da Vinci Eye](https://apps.apple.com/us/app/art-projector-da-vinci-eye/id6467633766)
* [6] [Simply Piano](https://apps.apple.com/us/app/simply-piano/id6503691129)