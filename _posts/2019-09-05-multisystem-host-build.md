---
layout: post
title: 黑苹果主机安装
subtitle: 安装 Mac OSX 10.14.6、Windows10 1903、Ubuntu 18.04.3 LTS 三系统主机
comments: true
---



前段时间把手头的 MacBook Pro 2019 卖掉了，手头现在的唯一的主力机是一台蓝天的准系统 P750TM ，一开始是想安装一台黑苹果主机算了，但是转念一想我确实还需要一台 Windows 的二奶机，又想到现在 Linux 相关的都开发还都是在主力机的虚拟机里面，所以干脆这台机子装三个系统算了。

 

## **一.硬件选择**

CPU： i9 9900K   散热：银欣AR11

内存：海盗船复仇者 16GX2 3000MHZ

主板：华擎 Z390 Phantome Gaming ITX/AC，带的无线网卡的型号为 INTEL AC9560

显卡：蓝宝石 RX580 2048SP 8G 白金版 （黑苹果刷 RX570 的VBIOS可以免驱动）

机箱：银欣（SilverStone）ML08B-H

电源：银欣SX500-LG

无线网卡：BCM94352Z  （考虑到黑苹果，这个垃圾网卡居然卖 300...，我特喵 INTEL AX200 才 60 块钱，把它按到地上摩擦。）

硬盘选择：

1.三星 970 EVO PLUS   1T    （安装 Ubuntu）

2.西部数据 SN750 黑盘  1T        （安装 Mac OSX）

3.三星 860 EVO 1T    （安装 Windows）

4.日立 1T 7200转 2.5寸机械硬盘，用于存放公用数据。

## 二.硬件安装

正常安装 *ML08B-H* 内部空间较大，相比于 *迎广肖邦*  简单许多，唯一需要注意的是  *蓝宝石 RX580 2048SP 8G 白金版*  的显卡高度与此机箱的显卡位高度差不多，你需要先将机箱附送的 PCI-E 延长板插到显卡上，向上扶着，再将 PCI-E 延长槽插在主板上，然后放下显卡让其插入延长槽。 

## 三.系统安装

配件都是陆陆续续到的，所以一开始是想把 MAC 和 Ubuntu 安装在机械里面然后在迁移过去的，但后来迁移的时候出了乱七八糟的问题，虽然都解决了三个系统能正常用了，但是还有一点小瑕疵就是不爽，强迫癌，刚好想起应该写一篇博客记录一下，所以除了 Windows 全部重新装了。

这里注意一下，建议安装 Ubuntu。

### 1.Ubuntu 18.03 LTS 安装 ###

去 Ubuntu 官网下载镜像。

通过 [Rufus](ftp://degage.tech/%E7%B3%BB%E7%BB%9F%E9%95%9C%E5%83%8F/rufus-3.4.exe) 工具制作好后，U 盘安装就行。注意勾选硬件专有驱动。

这里不知道为什么一打开 Monodevelop，CPU 风扇声音就巨大，但看 CPU 使用率和温度并不高，索性主板里面按温度限制转速了。


### 2.Windows 10 1903 安装

下载 Win10 1903 的ISO 镜像，解压至 U盘即可，插入任意 USB 接口正常安装即可。

在 Windows 环境下将 RX 580 2048SP VBIOS 刷写为 570 的，你可以在[此处](ftp://degage.tech/%E5%B7%A5%E5%85%B7/%E7%A1%AC%E4%BB%B6%E7%BB%B4%E6%8A%A4/rx570vbios.rom)获取 570 的 VBIOS 文件，[这里](ftp://degage.tech/%E5%B7%A5%E5%85%B7/%E7%A1%AC%E4%BB%B6%E7%BB%B4%E6%8A%A4/rx580vbios.rom) 是我备份的 580 的 VBIOS 文件。（注意这里提供的 VBIOS 文件都是针对于蓝宝石白金版的，是否适用于其他品牌型号未经验证）

### 3.Mac OSX 10.14.6 安装3

下载系统相关 DMG 文件，https://pan.baidu.com/s/1xG3BxqI9Fu413ZT37M-fUg 提取码：oqut

- 使用 [TranMac](ftp://degage.tech/%E5%B7%A5%E5%85%B7/%E9%BB%91%E8%8B%B9%E6%9E%9C%E7%9B%B8%E5%85%B3/TransMac12.1.zip)，制作安装镜像。

- 使用[此处](https://github.com/bydavy/EFI-ASRock-Z390-Phantom-Gaming)的 EFI 直接替换镜像中的 EFI 文件夹，可以省去我们很多寻找驱动的时间。（感谢这位玩家）

- 正常安装，进入系统后使用四叶草配置工具，挂在系统的 EFI 分区然后同样使用上一步的 EFI 文件替换系统原有 EFI 文件夹。

  

  安装黑苹果，如果硬件选择正确的话，比起安装 Windows 麻烦不了多少，剩下的参数在慢慢调整优化即可。

  目前使用中已知BUG，  *HDMI 音频输出无效。*

  
  
  
  
  
  
  
  
  