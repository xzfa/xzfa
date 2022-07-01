ubuntu下QQ, 微信, 讯雷, 网盘等无痛安装, 解决高分辨率问题hidpi
感谢
配置deepin-wine
配置方法
下载wine的容器
高分辨率问题
前提告知
解决高分辨率问题，开启Wine的hidpi
最后一步
完成
感谢
首先感谢 deepin团队为大家做出的贡献.
当然如果直接使用deepin(深度操作系统)大可不必这么麻烦. 不过试验室都是用的ubuntu.

配置deepin-wine
使用的配置地址
这里的wine和apt install 的wine是不一样的(个人猜测).
需要使用特殊配置过的wine.

配置方法
wget -qO- https://raw.githubusercontent.com/wszqkzqk/deepin-wine-ubuntu/master/online_install.sh | bash -e
1
下载wine的容器
容器也就是QQ, 微信等应用.
常用App下载地址
完整目录地址

下载完成后, 使用dpkg进行安装

sudo dpkg -i xxx.deb
1
高分辨率问题
高分辨率下显示过小问题

前提告知
安装完成后, 先运行一下.

比如对于QQ软件，在Ubuntu-Gnome（ubuntu18.04以后，都是定制的Gnome，但是看着像Unity），就是点击WIN键，在出现的界面，输入QQ，然后就会出现QQ图标。然后使用箭头选择QQ，然后回车。 这时候，对于4K屏幕来说， wine显示的QQ会特别的小，眼睛看着很累。 打开之后，就直接关闭QQ。
上一步不是没有作用的。通过上面的操作，这样在home目录（~/.deepinwine） 目录下就会生成一个容器文件夹（如果你是上一步是QQ， 那么一般就是“Deepin-QQ”文件夹）.
解决高分辨率问题，开启Wine的hidpi
下面一步在命令行中执行, 并且需要修改一下下面的命令 ，请看下面分析。

# 以QQ为例
```sh
WINEPREFIX=~/.deepinwine/Deepin-QQ  /usr/bin/deepin-wine  winecfg
```



```sh
WINEPREFIX=~/.deepinwine/<唯一需要修改的地方:你的APP目录> /usr/bin/deepin-wine  winecfg
```

上面命令执行 之后，会弹出
一个窗口。

最后一步
在弹出的窗口中, 最上方选择"Graphics". . 我是4k:3840x2160分辨率. screen resolution设置为150正好. 最好不要大于200. 会崩溃然后再也启动不了了.

**完成**
