---
title: Manjaro 安装记
date: 2016-06-23
tags:
- Linux
---
最近想尝试把 Android 的开发环境转移到 Linux 上, 因为不想继续在 Windows 使用 *babun* (经过订制的开箱即用的 cygwin) 凑合了. 我本着不折腾的原则, 来来去去安装了几个发行版本:
0. Ubuntu, 一直以简易好手上著称, 但是在我安装了最小版本之后发现双击 .deb 文件居然不能正常安装软件了, 安装搜狗输入法的时候也不正常. 遂放弃.
0. Deepin, 好评度不错的国内发行版本, 中文化很好, 特别是合作推出了很多国内软件的 Linux 版本, 想来对我这种小白应该很合适, 毕竟不想折腾只想安安静静写代码.
 但是安装之后发现桌面流畅程度真的是不敢恭维, 实在觉得卡了. 遂放弃.

最后看上了 ArchLinux, 但是安装过程比较繁琐, 我又不想折腾, 于是选择了基于 Arch 发行的 Manjaro.

Manjaro 安装很简单, 和 Ubuntu 等其他的桌面发行版本一样, 一路点点点就装好了. 不过装好之后还需要进行一些简单的配置.

# 更换源与添加源
```
#nano /etc/pacman.d/mirrors/China
[China]
Server = http://mirrors.ustc.edu.cn/manjaro/$branch/$repo/$arch

#nano /etc/pacman-mirrors.conf
OnlyCountry=China

pacman-mirrors -g


# /etc/pacman.conf 
[archlinuxcn]
SigLevel = Optional TrustedOnly
Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch

sudo pacman -Syy && sudo pacman -S archlinuxcn-keyring
```
不过在我添加了 archlinuxcn 的源之后安装 archlinuxcn-keyring 失败. Google之后得了解决办法:

```
pacman -Syu haveged
systemctl start haveged
systemctl enable haveged
rm -rf /etc/pacman.d/gnupg
pacman-key --init
pacman-key --populate manjaro
pacman-key --populate archlinuxcn
```

好了, 现在源配置好了, 安装 Chrome 和 Android-studio 都是一个命令的事了, 很爽!

# 安装 zsh
既然是用 Linux 当然没有忘记把 bash 换成 zsh
首先是安装 zsh:  
``` 
sudo pacman -S zsh 
```

接着配置 oh-my-zsh: 
``` 
sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)" 
```
最后更换默认的 shell: 
``` 
chsh -s /bin/zsh 
```
重启之后就就可以愉快的使用 zsh 了~

# 安装中文输入法
我选择的是安装搜狗拼音的 Linux 版本
```
sudo pacman -S fcitx-sougoupinyin
sudo pacman -S fcitx-im # 全部安装
sudo pacman -S fcitx-configtool # 图形化配置工具
```
之后就是还需要更改 **~/.xprofile**
```
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS="@im=fcitx"
```
最后在命令行输入 `fcitx` 就可以使用了
# 配置 Android 开发环境
Android-studio 在 archlinuxcn 源中有现成的包, 安装很简单就没什么好说的了.
不过在 Android-studio 安装好后, 当我想启动 AVD 时出现了错误, 不能启动. 好在也是经过 Google 之后解决了:
```
cd ～/Android/tools/lib64/libstdc++
mv libstdc++.so.6 libstdc++.so.6.bak
ln -s /usr/lib64/libstdc++.so.6 ～/Android/tools/lib64/libstdc++
```

经过如上步骤, 一个基础的 Android 开发环境就配置好了. 虽然只有上述 4 个简单的步骤, 但是还是折腾掉了我一个下午的时间, 所以想分享出来节省大家的时间. 经过一个下午的简单体验, 觉得 Manjaro 很适合新手使用, 有简单易用的图形化安装界面, 使得像我这样的小白也能轻易体会到 archlinux 的好处 (系统是滚动升级的, 软件包也都很新), 有 pacman 配合官方源和 archlinuxcn 源, 基本什么软件安装都是一行命令可以解决, 十分的爽快.