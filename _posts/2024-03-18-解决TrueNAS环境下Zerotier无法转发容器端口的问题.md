---
title: 解决TrueNAS环境下Zerotier无法转发容器端口的问题
date: 2024-03-18 02:24:00 +0800
categories: [网络, NAS]
tags: [Zerotier, TrueNAS, k8s]
pin: false
---

## 背景

一切的起因都在于我想要给我的NAS配备一个适合的相册服务。因为一些原因，近日从iPhone13 mini换成了OPPO Find X6，至此我已经先后使用过小米、苹果和OPPO三个生态系统，照片也就分散存储在各家的云服务上，如何有效地管理这些照片就成了一大难题。这个时候，我想起了我的NAS。它运行在TrueNAS Scale 23.10下，目前由3块4t硬盘组成RAID-Z5阵列，与Mac通过万兆交换机和雷电网卡连接。

在这之前，我访问NAS都是直接使用SMB连接，NAS的用途也仅限于文件的存储与备份，除了TrueNAS的管理界面外就没有其他的web需求了。在测试了Photoprism、Immich、Nextcloud等等私有云和私有相册管理工具之后，它们中没有一个能令我满意，Photoprism因为某些内部bug根本跑不起来，Immich的App会出现莫名其妙的卡登陆验证问题，Nextcloud本质上还是一个云盘，更多考虑的是对文件的处理，几乎没有任何与相册有关的功能。就在我放弃、甚至在思考迁移到黑群晖的前一晚，我在一个B站视频的评论区下了解到了[MT Photos](https://mtmt.tech)，一款国人开发的订阅制付费软件，可以免费试用一个月，而且有几乎是傻瓜式的TrueNAS配置方法，于是花了一晚上进行配置、同步照片，尽管它的AI识别准确性还有些欠缺，但是已经比其它工具要方便、稳定太多，于是我在第二天就直接购入了永久许可。

![image-20240318030911736](/assets/img/posts/image-20240318030911736.png)

但是，作为一个私有云相册，只能内网访问还不够，内网穿透还是要有的。我常用的内网穿透都是借助ZeroTier，因此我也顺理成章地尝试在TrueNAS下部署ZeroTier。ZeroTier在Truenas环境下有官方提供的Chart，但国内外的资料都很少，在排除了相当多的问题后，总算是让ZeroTier跑了起来，有关这些恐怕还得再写一篇文章来记录。

## 问题描述

我将NAS在ZeroTier的内网IP设置为`192.168.191.7`，直接访问这个IP（80端口）能够顺利跳转到TrueNAS后台，ssh功能也一切正常，但就在我想要通过28063端口访问相册服务的时候，却发现无法访问，wget显示connection refused，最最重要的是，似乎只有TrueNAS应用暴露的端口无法从外网访问，其它不依赖于这些的web服务则一切正常，看起来这个问题只和truNAS的所谓“应用”有关系。

![image-20240318022640489](/assets/img/posts/image-20240318022640489.png)

## 解决方法

它的核心其实就是一行命令，只不过在TrueNAS这一特殊环境下，我们还需要进行一些额外的操作。

```
iptables -t nat -A PREROUTING -i [ZeroTier网卡名] -p tcp -m tcp --dport [端口] -j DNAT --to-destination [本地IP]:[本地端口]
```

其中，中括号内是需要自行填充的内容，网卡名是指ZeroTier创建的虚拟网卡，它的名称是以`zt`开头的一串字符串，你可以在`Truenas -> 网络`一栏中找到它：

![3546660916](/assets/img/posts/3546660916.png)

`--dport`后的端口为通过ZeroTier访问时使用的端口，`--to-destination`后则需要填写在局域网内用于访问对应web服务的IP和端口，一般是本机IP地址和应用占用的端口。

> 如果你没有正确配置ZeroTier的话，ZeroTier每次重启都会重新生成新的MAC地址和对应的虚拟网卡，导致网卡名不断变化，请在正确配置ZeroTier之后再执行上述命令。
{: .prompt-warning }

最后，在ssh中执行该指令，确定有效之后，在`系统设置 -> 高级 -> 开机/关机脚本`中，选择`添加`，类型一栏选择`命令`，将上面的的命令粘贴进来，执行时间（什么时候）选择`初始化前期`，其余保持默认，点击`保存`即可。

<img src="assets/img/posts/1090611639.png" alt="1090611639" style="zoom:50%;" />

之后重启TrueNAS，等待相应服务正常启动之后便可以从外网正常访问服务端口了。

![image-20240318141547739](/assets/img/posts/image-20240318141547739.png)

## 原理解释

目前看来，这个问题应该与防火墙冲突有关。

TrueNAS的应用实际上基于Kubernetes，由于首字母k和尾字母s之间有8个字母，一般也会简写为k8s或kube，只不过TrueNAS部署的是更加精简的k3s版本。k8s的本质上是一个容器编排平台，可以让诸如Docker之类的容器以更加自动化、便利的方式部署。k8s的最小单位是容器集（Pod），内部可以包含多个容器，pod间的通信依赖于内网，因此k8s拥有一套自己的防火墙，而web服务则需要经过防火墙的转发才能正常通信，一般会给不同的应用分配不同的端口，再将这个端口映射到本地网络以供访问。k8s的防火墙基于系统防火墙，对于TrueNAS来说就是iptables，如果你查看iptables的filter表，可以看到大量由k8s为内网自动创建的路由规则。（相关内容与结构图来自于[RedHat](https://www.redhat.com/zh/topics/containers/what-is-kubernetes)）

![kubernetes_diagram-v3-770x717_0_0_v2_0](/assets/img/posts/kubernetes_diagram-v3-770x717_0_0_v2_0.svg)

另一方面，ZeroTier的原理是创建一块虚拟网卡，通过这个网卡转发流量，而TrueNAS上的ZeroTier同样基于k8s运行，这就意味着ZeroTier一定晚于系统防火墙启动，其自身的防火墙优先级也就更低，因此即使在内网可以访问的端口，也不一定可以被ZeroTier访问，因此理论上如果让防火墙的优先级低于ZeroTier，这个问题是可以解决的（参考[这里](https://www.bilibili.com/read/cv31416305)），但是TrueNAS为了稳定性禁用了很多功能，而且对系统的更改几乎都无法在系统更新中被保留，因此这条路走不通，便转而考虑为ZeroTier加一条nat规则，将来自ZeroTier的流量转发给本地端口：

```
iptables -t nat -A PREROUTING -i [ZeroTier网卡名] -p tcp -m tcp --dport [端口] -j DNAT --to-destination [本地IP]:[本地端口]
```

其中，`-t`指定使用nat表，`-A`指定这条规则追加在`PREROUTING`链后（对数据包做路由选择前执行的规则），`-i`指定包来源为ZeroTier的虚拟网卡，`-p`指定匹配TCP协议，并用`-m`指定使用iptables的TCP扩展模块；`--dport`匹配目标端口，`-j`表示匹配成功后触发DNAT，即目标地址转换，将目标地址转换为`--to-destination`中的地址。虽然这里的地址可以直接指向容器所在的ip地址，但是为了保证防火墙有效还是只转发到本地即可。

然而，正如刚才提到的，TrueNAS禁用了很多功能，其中就包括iptables-services，因此直接用ssh执行这一命令并不是一直有效的，只要重启就会被自动清除，即使通过`crontab -e`将它写入开机命令，下一次系统更新时也会被清除，而且会极大地影响系统稳定性，因此根据TrueNAS的文档，最佳的做法还是将其添加进TrueNAS系统的开机脚本。至此，相关问题得到了完美解决。

## 参考链接

1. [外部无法访问docker映射的端口](https://www.jianshu.com/p/9fdc78a774ac)
2. [Save custom iptables entry to config database?](https://www.truenas.com/community/threads/save-custom-iptables-entry-to-config-database.103874/)
3. [什么是 Kubernetes？](https://www.redhat.com/zh/topics/containers/what-is-kubernetes)
