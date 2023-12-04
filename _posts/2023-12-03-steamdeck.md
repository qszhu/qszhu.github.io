---
layout: post
title: Steam Deck近一年的使用体验
date: 2023-12-03 20:30:00 +0800
tags: [steamdeck, remote_desktop]
---

最近Steam Deck有了改款,有朋友问起是否值得入.查了下购物记录,我入手Steam Deck也有将近一年时间了.在此总结一下,希望能帮助到有需要的人.

## 关于容量

我的Steam Deck是初版的64G版本,自行更换了1T的SSD[1]+1T的SD卡.即便如此也没能塞下Steam库中已购入的,标记为适合在Steam Deck上游玩的游戏.囤积癖玩家可以参考下选择合适的存储容量.

## 关于游戏

Steam Deck的游戏模式类似于PC端的大屏幕模式.因为是Linux系统,需要通过WINE来运行Windows游戏.实测能够流畅运行部分3A游戏.

此外得益于Steam Deck的性能,也能够流畅运行不少模拟器.如RetroArch,Cemu,RPCS3,yuzu等.贴张图,就不展开了.

![Steam Deck](/assets/images/2023-12-03/sd.jpg)

对于小朋友爱玩的MC也有几个启动器,流畅运行更是不在话下.

## 关于系统

我日常使用大部分时间都是用的桌面模式.通过USB Dock可以外接显示器和键盘鼠标网线等.操作系统据说基于Arch,有pacman可以用.但为了安全/方便恢复,系统分区默认是只读的.也可以把pacman包安装到用户分区,但前提是系统依赖版本没有大的变化.我在这样正常使用大半年后遇到了Steam Deck更新了KDE的大版本,需要重新搞一遍依赖的问题.最终还是觉得太麻烦而放弃了这种方案.

此外官方支持安装windows[2],也说未来会支持双系统,不过目前似乎还没有官方支持的双系统方案.倒是网上很早就有很多安装双系统进而解锁XGP的教程.我一直没有机会尝试.

## 关于软件

Steam Deck上推荐的软件安装方式是通过flatpak.这是一种旨在减少Linux生态中软件包维护者重复工作的沙盒技术.虽然包总数还比较少,但覆盖了常用的软件,可以在这里[3]搜一下想用的软件有没有.

flatpak默认会安装和升级到最新的版本.想回退到某个历史版本的话可以参考文档[4].

### tmux

日常工作离不开tmux,但在Steam Deck上退出终端tmux就会被杀掉.一个实测可用的方法是[5]:

```
$ systemd-run --scope --user tmux
```

## 关于远程桌面

遇到不支持的软件或是Steam Deck的性能不足以运行的软件,就轮到远程桌面出马了.flathub上可以找到的Remmina和KRDC都支持VNC和RDP协议,连接Windows和Linux都可以,推荐日常使用.

由于我的开发机是Mac,有一些额外的问题.Mac虽然支持VNC协议,但由于VNC协议通常是直接传输屏幕像素的,效率比较低.再加上Mac的Retina分辨率,数据量还要翻两番.这就造成了使用VNC时视觉上的卡顿.Mac App Store上有苹果官方的收费软件Apple Remote Desktop,支持自适应画质,使用体验不错,但仅能在Mac上运行.

在flathub上能找到的其他免费远程桌面软件中,AnyDesk使用起来最方便.但在Mac上需要给予录屏和辅助功能等的诸多权限,猜测是通过私有协议传输画面的.在使用了一段时间后AnyDesk上出现了提示购买的消息.看了下其使用协议,日常工作使用不在免费使用的范畴,需要购买许可.如果不差钱的话还是可以推荐购买的.

另一个选择是(名字平平无奇的)Remote Desktop Manager.上手配置有些繁琐,但其支持的协议中包含Apple Remote Desktop(ARD),使用体验也接近官方的软件.但可能由于是第三方实现的原因,早期使用时在进行拖拽操作时经常闪退.不过随着最近的几次版本更新似乎稳定了许多,可以基本满足日常需求了.

### VPN

原始的VNC协议传输的是非加密数据,这意味着明文数据会被网络上的中间人看到.此外跨局域网的使用还需要解决打洞或是流量中转的问题.这可以通过设置VPN来解决.Steam Deck的系统支持Wireguard,在网络设置中直接新建连接即可.

### 画质

为减少数据传输,远程画面可能会被压缩.对画质要求高的工作如美术设计等,就不适合使用远程桌面.

### 隐私

即便通过网络传输的数据已经过加密来防止网络上的中间人偷窥,远程桌面软件本身还是可以看到你的屏幕画面的.使用与否就取决于你对远程桌面软件的信任了.这也是我尝试自行实现一个VNC客户端的原因,下次可以详细说说VNC的实现.

## 总结

个人感觉Steam Deck的便携性是优于笔记本的.作为PC的性能可满足日常使用,而更专业的使用则可通过远程桌面来实现.要说为什么需要一台更便携的PC,可以追溯到我近10年以前的探索:

![远程运维](/assets/images/2023-12-03/remote.jpg)

图中从左到右分别是:移动热点+充电宝,树莓派,运行着远程桌面连接到树莓派的手机,无线键盘.借助这些在不带笔记本外出的情况下处理on-call.Steam Deck则为这个思路提供了新的平台.

相信大家已经见过不少坐在路边/站在垃圾桶旁处理on-call的挨踢打工人了.下一阶段应该也已经被考虑好了,那就是在车上😛:

![远程会议](/assets/images/2023-12-03/car.jpg)

本文在Steam Deck上通过远程桌面完成.

## 参考

* [1] [Steam Deck SSD Replacement](https://www.ifixit.com/Guide/Steam+Deck+SSD+Replacement/148989)
* [2] [Steam Support :: Steam Deck - Windows Resources](https://help.steampowered.com/en/faqs/view/6121-ECCD-D643-BAA8)
* [3] [Flathub - Apps](https://flathub.org/en)
* [4] [Tips and Tricks - flatpak/flatpk wiki - Github](https://github.com/flatpak/flatpak/wiki/Tips-&-Tricks#downgrading)
* [5] [sshd - tmux session killed when disconnecting from ssh - Unix & Linux Stack Exchange](https://unix.stackexchange.com/a/318413)
