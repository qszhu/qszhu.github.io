---
layout: post
title: 即插即用电视棒
date: 2021-04-18 15:30:00 +0800
tags: [rpi, kodi]
---

## 1. 背景

小朋友最近喜欢看粉红猪一家的动画片。家中电视里预装了一些视频网站的TV版本，不过有几个问题：

* 每集中间都有广告。要知道一集才5分钟，广告就要30秒
* 内容不全，看到后面就要充VIP。而且不知道充了以后会不会有VIP专属广告
* 操作麻烦。有时候家里只有老人在，想开视频却不知道要点哪些地方

本来也不是不能看，就没太在意。直到有一天孩子他妈给娃买了早教体验课，要我每天督促小朋友学习，搞得我一脸懵逼。原来孩子他妈在陪娃看片的时候，已经被中间插播的广告成功转化了。敢情我一直盯着自家产品的转化漏斗，结果家人却先被别人家的产品给转化了。郁闷之余，我开始着手解决这个问题。

## 2. 分析

内容的问题相对好解决，就不赘述了。如果能够播放指定内容的话，广告问题也就解决了。现在的电视插个U盘/移动硬盘就可以播放其中的内容，不过操作上需要这里点点那里点点，还是比较麻烦。理想情况是开机播片，无需其他操作。受到打开NS电视就会自动开机并切换到NS输入源的启发，明白了其实需要的是一个HDMI输入源，类似电视棒。然而看了下市面上仅有的几款电视棒，似乎没有满足要求的。于是只能DIY。

## 3. 过程

回收利用了一个多年前用来做NAS的rpi3，烧了Kodi，拷贝了视频。原本买rpi的时候附送了一根带开关的充电线，于是就能实现物理按键开关机了。剩下的就是实现开机后自动播放了。

根据文档[1]，在`~/.kodi/userdata/autoexec.py`中执行开机命令

```python
import xbmc
xbmc.executebuiltin( "PlayMedia(/storage/.kodi/userdata/playlists/video/peppapig.m3u)" )
xbmc.executebuiltin( "PlayerControl(RandomOn)" )
xbmc.executebuiltin( "PlayerControl(RepeatAll)" )
```

注意烧录镜像中的路径有些区别，是`/storage/.kodi/userdata/autoexec.py`。

测试，按下开关，电视机自动开机并切换输入源，开始自动随机播放所有视频。测试成功。

## 4. 其他

了解了一下`Kodi`项目，原名是叫`XBMC`的，所以配置里还保留了这个名字；插件似乎很丰富[2]；还可以安装游戏模拟器[3]，貌似插个手柄就能在电视上打游戏；还有一个手机app[4]，可充当遥控器使用。

下一步准备看下IPTV是否也能用这种方式搞定。电信的IPTV各种反人类，想要看个电视先要启动个半天，再沿着一条很深的路径点个7、8下。而且默认的扣费方式是直接走电信账单，被不明所以的老人小孩点几下可能就扣了钱。关闭扣费的方式则是在另一个很深的入口里打开某个选项（别问我是怎么知道的）。希望能在像这次一样造成不可挽回的结果之前搞定吧……

## 参考资料
* [1] [List of built-in functions - Official Kodi Wiki](https://kodi.wiki/view/List_of_built-in_functions)
* [2] [Add-ons \| Kodi \| Open Source Home Theater Software](https://kodi.tv/addons)
* [3] [Game client \| Kodi \| Open Source Home Theater Software](https://kodi.tv/addons/game-client)
* [4] https://apps.apple.com/us/app/official-kodi-remote/id520480364
