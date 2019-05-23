---
layout: post
title: 如何科学上网
subtitle: 自主搭建科学上网的服务器
---
#### 最近给小伙伴们提供的科学上网服务器被墙的厉害，用的人多也容易被检测出来，不停的创建新实例，换地址，累觉不爱~~~所以写下方法，授人以鱼不如授人以渔，希望大家可以建一台自己专属的科学上网服务器。

* ##### 拥有一台科学上网服务器
 首先我们提供科学上网服务的服务器本身应该要可以科学上网的，这里我选择了在 [Vultr](https://www.vultr.com/?ref=7529541 "无耻的推广") 上购买一台服务器（可以支付宝支付）， 5$/月，线路一般东京的比较快些。  
系统的话我选择了 [Ubuntu 16.04](http://www.ubuntu.org.cn/download "获取 Ubunutu") 。

* ##### 远程服务器

 从 Vultr 上查看自己购买服务器的账户和密码，Servers 下面选择 Instances 页，点击具体的服务器实例，应该可以看到如下的信息：

![Vultr 服务器信息示例](/img/posts/science-internet/server-example.png "Vultr 服务器信息示例")

 通过 [XShell](https://www.netsarang.com/en/xshell/) (个人使用是免费的) 远程服务器，当然你也可以通过其他方法。

* ##### 安装并配置 Shadowsocks ：
``` shell
  apt update  
  apt install shadowsocks
```
 安装完毕后配置 Shadowsocks，新建一个配置文件，名称随意
 ``` shell
   vim /etc/shadowsocks.json
```
 内容如以下格式（多端口配置，密码、端口）：
 ``` shell
   {
    "server":"0.0.0.0",
    "port_password": {
    "8381": "932444208",
    "8382": "932444208"
    },
    "timeout": 300,
    "method": "aes-256-cfb",
    "password":"932444208"
 }
```
 保存后，启动 SS 服务器，命令如下：
  ``` shell
   ssserver -c /etc/shadowsocks.json start
```
OK，到这里科学上网服务器就搭建好了~~

* ##### 使用科学上网客户端：

 可以通过此页面了解并下载 [Shadowsocks](http://shadowsocks.org/en/download/clients.html) 与个平台相关的客户端。

 * ##### 科学上网服务优化：
   * XShell 会话退出后，SSServer 进程退出，客户端无法科学上网，解决办法如下：
        ``` shell
        nohup ssserver -q -c /etc/shadowsocks.json &
        ```
        此指令格式：   nohup 你的指令 &  

    * 使用锐速加速科学上网服务器（一键脚本，网上搜集的，有一定效果）
        ``` shell
        wget xiaofd.github.io/ruisu.sh && bash ruisu.sh
        ```
    * 开启BBR  
        linxu 内核低于 4.9 需要更新内核   

        查看内核版本
        ``` shell
         uname -r
        ```
        安装 Hardware Enablement Stack (HWE)，更新内核
        ``` shell
         apt install --install-recommends linux-generic-hwe-16.04
        ```
        删除旧内核（可选的，慎用）
        ``` shell
         apt autoremove
        ```
        重启

        查看是否已启用 BBR
        ``` shell
         lsmod | grep bbr
        ```
        若无，则修改 sysctl.conf
        ``` shell
         echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
         echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
         sysctl -p
        ```
    * 科学上网服务开机启动  

         将服务启动命令添加至  /etc/rc.local 文件
        ``` shell
          nohup ssserver -q -c /etc/shadowsocks.json &
        ```
* ##### 科学上网服务被墙

   可 Ping 但是 XShell 无法连接。  

   国内使用端口扫描测试，22 端口关闭

      测试地址：http://tool.chinaz.com/port  

   国外使用端口扫描测试，22 端口开启

      测试地址：https://www.yougetsignal.com/tools/open-ports  

   那不用想了，很大程度上被墙了，更换服务器吧。

* ##### 科学上网服务花式使用：

  [SSTAP 游戏代理工具](https://www.sockscap64.com/sstap-homepage-chinese-traditional/)