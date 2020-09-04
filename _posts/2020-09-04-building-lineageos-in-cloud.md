---
layout: post
title: Android备用机方案
date: 2020-09-04 21:30:00 +0800
tags: [android, open source, lineageos, essential phone]
---

### 1. 背景

前一阵因为tiktok的事情又涌现出一拨节奏大师，于是就有了这么一出：

![立个flag：如果iPhone被封杀我就去做开源Android手机🤪](/assets/images/2020-09-04/1.jpg)

虽说我不觉得iPhone会被封杀，但一些朋友提醒我不要把话说得太死。于是就当是为了应对最坏的情况作的准备吧🤪

### 2. 硬件

首先是选手机。看下来一加似乎对开发者比较友好，而以发烧起家的小米似乎需要强制插SIM卡激活，就显得不是很方便。不过如今国产手机都开始走高端路线，低端机也都要千元起跳了。要降低成本，要么选二手设备，要么选没有电话功能的平板。

二手水深，国内的设备不敢碰，于是目光转向洋垃圾。成功锁定Essential Phone，某宝关键字`安卓之父手机`。一台成色不错的手机500大洋搞定✌️

平板方面则无意中发现了可能是最便宜的安卓平板，酷比魔方的某款7寸平板，据说还能打电话，具体型号可以自行搜索。不过似乎没找到有解锁设备的成功案例，就排除掉了。

### 3. 软件

尽管Essential Phone的公司已经倒闭，但ROM还能找到[1]。从Android 7到Android 10，API level覆盖了25-29，拿来当开发机也是极好的。据说未来还可以升上Android 11。

#### 3.1 刷机

刷机的时候遇到点问题。用最新版的`platform-tools`刷机时总会报错，似乎是因为现在Android机引入A/B分区的关系。使用ROM发布时对应的`platform-tools`版本就可以了。例如，最新的稳定版ROM是在2020年2月发布的[2]：

```
February 2020 Release | Q 10 - Q Release 6 - QQ1A.200105.032 | All
...
```

而2020年2月发布的`platform-tools`的版本是`29.0.6`[3]：

```
29.0.6 (February 2020)
  * adb
    * ...
```

因为`platform-tools`的下载页面只提供了最新版本的下载链接，所以下载地址就需要手动调整下：

```
http://dl.google.com/android/repository/platform-tools_r29.0.6-darwin.zip
```

#### 3.2 系统编译

因为拒绝各类全家桶（连Google全家桶也不想要），所以要刷AOSP第三方ROM。第三方ROM首选CyanogenMod，支持的设备多。国内外各种官方非官方ROM大都是照着它改的。但似乎前几年看到新闻说项目终结了（残酷的世界）。查了下现在新的项目叫LineageOS[4]，支持的设备里包含了Essential Phone[5]。

Lineage项目有非常友好的文档[6]，一般情况下照做就可以了。不过Android系统对编译机的要求比较高。当年做定制ROM的团队都会有一台性能强劲的工作站来打包，即便放在今天性能也远超家用机。不过好在如今云服务蓬勃发展，个人也可以用低廉的价格租用虚拟云服务器。

#### 3.3 云编译

时间有限，我只在腾讯云上做了测试。选了16核32G内存的竞价实例，每小时费用不到1块钱。虽说文档上写8G内存100G硬盘就可以，但实测下来记录到20G内存和200G硬盘的使用。云硬盘选择挂载新的SSD数据盘，这样即便退还实例，已经同步过的代码也可以保存下来，方便下次使用。

网络也是需要解决的问题。在大陆的服务器上以几K的速度同步几十G的代码是不现实的。尝试过清华的镜像[7]，但代码同步有问题。最终用了香港的实例，价格稍微贵一点，但代码得以顺利同步。

在编译之前还需要提取手机中的专有文件。这也是开源的Android系统一直无法绕过的坎。因为云服务器无法连接本地的手机（就算技术上可行可能也会很复杂），就尝试从打包文件中提取[8]。不过试了下Lineage自己的包是不行的，仍旧少文件；Essential Phone官方的包没有测试。直接在Github上搜索`proprietary_vendor_essential`就可以得到需要的文件了。

编译，刷机：

![新鲜出炉](/assets/images/2020-09-04/2.jpg)

干净的系统，待机一周无压力：

![超长待机](/assets/images/2020-09-04/3.jpg)

从买云服务器到编译完成，大约2个小时就可以。花费5块钱左右。

### 4. 未来改进

* 云硬盘不挂载可以做成快照，收费更低，并且腾讯云有50G的快照免费额度。不过按照现在AOSP的大小可能很难缩到这个范围之内。可以尝试不同步代码历史(`--depth=1`)，以及去掉当前设备以外的设备所需要的文件；
* 可使用`ccache`提升编译速度[9]，代价是占用更多的磁盘空间；
* 尝试使用更多核心的云服务提速；
* 编译前的准备工作可以交给自动化脚本（如`ansible`），进一步缩短时间。

### 参考资料
* [1] [Essential Backup](https://genericbleach.github.io/EssentialBackup)
* [2] [Current Builds - Developer Page - Formstack](https://genericbleach.github.io/EssentialBackup/Current%20Builds.html)
* [3] [SDK 平台工具版本说明  \|  Android 开发者  \|  Android Developers](https://developer.android.com/studio/releases/platform-tools.html)
* [4] [LineageOS – LineageOS Android Distribution](https://lineageos.org/)
* [5] [LineageOS Downloads](https://download.lineageos.org/mata)
* [6] [Build for mata \| LineageOS Wiki](https://wiki.lineageos.org/devices/mata/build)
* [7] [lineageOS \| 镜像站使用帮助 \| 清华大学开源软件镜像站 \| Tsinghua Open Source Mirror](https://mirrors.tuna.tsinghua.edu.cn/help/lineageOS/)
* [8] [Extracting proprietary blobs from LineageOS zip files \| LineageOS Wiki](https://wiki.lineageos.org/extracting_blobs_from_zips.html)
* [9] [Build for mata \| LineageOS Wiki](https://wiki.lineageos.org/devices/mata/build#turn-on-caching-to-speed-up-build)
