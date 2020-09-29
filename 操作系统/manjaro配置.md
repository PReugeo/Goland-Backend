# 系统安装

## 系统下载

首先去官网下载 manjaro 镜像，选择 kde 桌面的镜像，然后下载 iso 文件



## 刻录 U 盘

准备一个 8 g 左右大小的 U 盘，下载刻录工具 **rufus** 



## 安装系统

制作完启动盘后，进入 BIOS 修改启动项，进入 manjaro 进行安装



# 系统配置

## 初始配置

首先更换国内镜像源

```shell
sudo pacman-mirrors -i -c China -m rank
```

选择一个延迟最低的，然后添加 CN-Arch 源

```shell
sudo pacman -S vim
sudo vim /etc/pacman.conf
# 在文件最后两行加入
[archlinuxcn]
SigLevel = Optional TrustedOnly
Server = https://mirros.tuna.tsinghua.edu.cn/archlinuxcn/$arch
```

然后更新系统源

```shell
sudo pacman -Syy
sudo pacman -S archlinuxcn-keyring
sudo pacman -Syu
```

安装搜狗输入法

```shell
sudo pacman -S fcitx-im fcitx-configltool manjaro-asian-input-support-fcitx
sudo pacman -S fcitx-sogoupinyin
```

然后在 `~` 目录下新建 .xprofile 文件

```shell
sudo vim ~/.xprofile
# 加入以下几行
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS="@im=fcitx"
```

安装 vscode 和网易云和 yay

```shell
sudo pacman -S visual-studio-code-bin
sudo pacman -S netease-cloud-music
sudo pacman -S yay
```

安装 tim 和配置输入中文

```shell
sudo pacman -S deepin.com.qq.office
# 必要依赖 gnome-daemon
sudo pacman -S gnome-settings-daemon
# 配置输入中文
cd /opt/deepinwine/tools
sudo vim run.sh
export GTK_IM_MODULE="fcitx"
export QT_IM_MODULE="fcitx"
export XMODIFIERS="@im=fcitx"
```

设置 `gnome-settings-daemon`  自启动

```rust
系统设置->开机或关机->自动启动->添加脚本->输入/usr/lib/gsd-xsettings
```

## 配置 zsh

```shell
sudo pacman -S zsh
```

下载 oh-my-zsh 

首先配置 hosts 文件

```shell
151.101.76.133 raw.githubusercontent.com
```



安装 oh-my-zsh

```shell
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

 安装插件

zsh-asutosuggestions

```shell
git clone git://github.com/zsh-users/zsh-autosuggestions $ZSH_CUSTOM/plugins/zsh-autosuggestions
```

zsh-syntax-highlighting

```shell
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

```zsh
plugins=(
	git
	zsh-autosuggestions
	zsh-syntax-highlighting
)
```



autojump

```shell
sudo pacman -S autojump
echo "[[ -s /etc/profile.d/autojump.sh ]] && source /etc/profile.d/autojump.sh" >> ~/.zshrc
```

然后运行 `source .zshrc` 即可