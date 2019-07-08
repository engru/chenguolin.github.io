---
layout:  post  # 使用的布局（不需要改）
catalog: true  # 是否归档
author: 陈国林 # 作者
tags:          #标签
    - 工具
---

Mac OSx环境下使用最多的是iTerm2 + Oh My Zsh，两者结合可以打造一个无比强大的终端体验

# 一. iTerm2
官网: https://www.iterm2.com/
安装下载: https://www.iterm2.com/downloads.html

安装iTerm2的过程会自动安装`zsh`安装路径在`/bin/zsh`，由于Mac默认使用`dash`我们需要修改默认终端使用`zsh`

1. dash->zsh: `$ chsh -s /bin/zsh`
2. zsh->dash: `$ chsh -s /bin/bash`

# 二. Oh My Zsh
开源地址: https://github.com/robbyrussell/oh-my-zsh
安装: `sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"`

# 三. 配置
1. 安装 PowerLine  
   `$ pip install powerline-status --user`
2. 安装 Themes
   ```
   mkdir tmp
   cd tmp
   git clone https://github.com/fcamblor/oh-my-zsh-agnoster-fcamblor.git
   cd oh-my-zsh-agnoster-fcamblor/
   ./install   //安装完成后会拷贝到oh my zsh的themes目录~/.oh-my-zsh/themes
   ```
3. 安装 Highlighting
   ```
   cd ~/.oh-my-zsh/custom/plugins/
   git clone https://github.com/zsh-users/zsh-syntax-highlighting.git
   ```
4. 配置 ~/.zshrc
    * 配置theme: ZSH_THEME="ys"
    * 配置plugin: zsh-syntax-highlighting加在plugins括号后面
    * 文件的最后一行添加：source ~/.oh-my-zsh/custom/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
5. 配置文件生效  
   `$ source ~/.zshrc`

# 四. 效果
![](https://github.com/chenguolin/chenguolin.github.io/blob/master/data/image/iTerm-zsh.png?raw=true)

