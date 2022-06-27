---
title: terminal配置git
tags: zsh、git
categories: 机器配置
date: 2022-06-27 13:14:47
---

老话说在前头：工欲善其事，必先利其器。想要玩转 macOS 的终端，想要用的顺手、看着高端，折腾是必不可少的。

### **第一步，安装 [HomeBrew](https://brew.sh/index_zh-cn.html)**

作为 macOS 必备的包管理工具，相信大家肯定已经很熟悉了，没安装的朋友可以执行下面命令装一下，安装过的可以执行下面命令可以进行更新。

```bash
/usr/bin/ruby -e "$(curl -fsSL <https://raw.githubusercontent.com/Homebrew/install/master/install>)"
```

### **第二步，更新 zsh、git**

macOS 一般会自带 zsh，不过版本会比较早，我们先更新一下，以便使用最新特性。

```bash
brew install zsh

==> Downloading <https://homebrew.bintray.com/bottles/zsh-5.7.1.high_sierra.bottle.tar.gz>
######################################################################## 100.0%
==> Pouring zsh-5.7.1.high_sierra.bottle.tar.gz
/usr/local/Cellar/zsh/5.7.1: 1,515 files, 13.3MB
```

### **第三步，切换至 zsh 并安装 oh-my-zsh**

查看当前使用的 shell

```bash
echo $SHELL

/bin/bash
```

查看安装的 shell

```bash
cat /etc/shells

/bin/bash
/bin/csh
/bin/ksh
/bin/sh
/bin/tcsh
/bin/zsh
```

切换为 zsh

```bash
chsh -s /bin/zsh
```

重启终端即可使用 zsh。

接下来安装 oh-my-zsh

```bash
sh -c "$(curl -fsSL <https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh>)"
```

安装完成后，终端展示如下内容：

```bash
  ____  / /_     ____ ___  __  __   ____  _____/ /_
 / __ \\/ __ \\   / __ `__ \\/ / / /  /_  / / ___/ __ \\
/ /_/ / / / /  / / / / / / /_/ /    / /_(__  ) / / /
\\____/_/ /_/  /_/ /_/ /_/\\__, /    /___/____/_/ /_/
                        /____/                       ....is now installed!

Please look over the ~/.zshrc file to select plugins, themes, and options.

p.s. Follow us at <https://twitter.com/ohmyzsh>.

p.p.s. Get stickers and t-shirts at <http://shop.planetargon.com>.
```

### **第四步，配置 oh-my-zsh**

看到这里，安装流程已经完毕啦，执行最后的配置，就可以进行体验了。

打开 oh-my-zsh 配置文件

```bash
# 打开 zshrc 文件进行编辑，也可以使用 vim 编辑器
open ~/.zshrc
# 本人使用的是 vs code
open ~/.zshrc -a Visual\\ Studio\\ Code
```

**主题**

配置项 `ZSH_THEME` 即为 oh-my-zsh 的主题配置，oh-my-zsh 的 GitHub Wiki 页面提供了 [主题列表](https://github.com/robbyrussell/oh-my-zsh/wiki/themes)当设置为 `ZSH_THEME=random` 时，每次打开终端都会使用一种随机的主题。

**插件**

```bash
plugins=(git osx autojump zsh-autosuggestions zsh-syntax-highlighting)
```

**注意：**其中 zsh-autosuggestions 和 zsh-syntax-highlighting 是自定义安装的插件，需要用 git 将插件 clone 到指定插件目录下：

```bash
# 自动提示插件
git clone <https://github.com/zsh-users/zsh-autosuggestions.git> $ZSH_CUSTOM/plugins/zsh-autosuggestions
# 语法高亮插件
git clone <https://github.com/zsh-users/zsh-syntax-highlighting.git> $ZSH_CUSTOM/plugins/zsh-syntax-highlighting
```

需要其他插件的可以自行安装，如果插件未安装，开启终端的时候会报错，按照错误提示，安装对应的插件即可。

更新配置

```bash
source ~/.zshrc
```

更新完配置即可生效，不想更新配置的话，新开一个终端同样可以生效。

正所谓风雨之后见彩虹，经过这一番捣鼓，电脑用起来更加顺手了，可以愉快的开发了。
