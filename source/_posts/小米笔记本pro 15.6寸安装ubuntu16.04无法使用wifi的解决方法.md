title: 小米笔记本pro 15.6寸安装ubuntu16.04无法使用wifi的解决方法
date: 2018-01-06 15:21:58
tags:
	- 技术相关
---


前不久买了台15.6寸的小米pro,顺利安装win10、ubuntu 16.04双系统之后发现ubuntu的wifi一直不能启动。

在小米社区上搜索找到的解决方法基本都是在/etc/modprobe.d/blacklist.conf文件后面加上acer-wmi或者在/etc/modprobe.d/ 下创建文件 xiaomi.conf并写入blacklist acer_wmi。

据说在小米13.3上面是可行的。但是我这边一直失败。而且也找不到15.6寸的小米pro解决方法。难道是还没有人用15.6寸的小米笔记本去装ubuntu 16.04吗？

# 原因

折腾了好几天,终于在ubuntu社区上找到解决方法。这里做下笔记,希望可以让有需要的人看到。

社区上具体的回答可以在[这里](https://askubuntu.com/questions/910934/intel-wifi-card-not-recognised-in-ubuntu-16-04)查看。


# 解决方法

chili555大神给出的解释是Ubuntu 16.04没有覆盖8086:24fd这个wifi设备的驱动,需要自己安装。或者直接安装Ubuntu 17.04,这个版本的ubuntu已经覆盖了这个驱动。


自己安装wifi驱动的方法也很简单,首先下载下面链接的软件:

- [linux-firmware_1.169_all.deb](https://github.com/bluesky466/bluesky466.github.io/blob/develop/source/wifi/linux-firmware_1.169_all.deb)
- [linux-headers-4.10.14-041014-generic_4.10.14-041014.201705031501_amd64.deb](https://github.com/bluesky466/bluesky466.github.io/blob/develop/source/wifi/linux-headers-4.10.14-041014-generic_4.10.14-041014.201705031501_amd64.deb)
- [linux-headers-4.10.14-041014_4.10.14-041014.201705031501_all.deb](https://github.com/bluesky466/bluesky466.github.io/blob/develop/source/wifi/linux-headers-4.10.14-041014_4.10.14-041014.201705031501_all.deb)
- [linux-image-4.10.14-041014-generic_4.10.14-041014.201705031501_amd64.deb](https://github.com/bluesky466/bluesky466.github.io/blob/develop/source/wifi/linux-image-4.10.14-041014-generic_4.10.14-041014.201705031501_amd64.deb)


然后执行命令将它们全部安装,接着重启电脑就可以了:

```
sudo dpkg -i *.deb
```

## 关闭secure boot

这个步骤可能不需要。因为我之前在搜索其他解决方式的时候就已经将它关闭了

如果你按照我上面的方法安装驱动之后还是不能使用wifi的话。进入bois查看是否位于UEFI且开启了secure boot,是的话将它关闭。

小米笔记本进入bois的方法是开机的时候狂按F2。进入之后找到secure boot关闭它就可以了。
