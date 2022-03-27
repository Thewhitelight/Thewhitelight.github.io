---
title: Mac 装机配置
date: 2022-03-27 10:20:20
tags:
    - mac 
categories:
    - mac
    - soft
---
# 日常装机软件

## 系统工具类:

| 名称  | 说明 |
| :----: | :----: |
| Go2Shell | 在 Finder 里快打开当前目录的 Terminal 窗口 |
| ClashX | 代理工具 |
| Alfred4 | 应用快速启动工具,不止于此|
| AppleCleaner| 卸载软件,可清空软件全部目录|
| iTerm | 终端工具 |
| The Unarchiver| 解压软件 |
| CheatSheet | 查看软件快捷键 |
| Rectangle | 分屏软件 |
| Plash | 将网页变成桌面 |
| MenubarX | 菜单栏打开网页 |
| AltTab | 切换窗口,支持各种自定义 |
| BLEUnlock | 绑定手机根据蓝牙信号锁屏 |
| HandShaker | 操控 Android 手机文件 |
| PopClip | 划词扩展工具 |
| PopMaker | 制作 PopClip 扩展的工具 |
| Bob| 翻译软件,搭配 PopClip |
| Rayon | 服务器监控软件 |

## Chrome App:

| 名称  | 说明 |
| :----: | :----: |
| regex101 | 测试正则表达式软件 | 
| Hoppscotch | 测试接口软件 |

<!-- more -->

# 配置:
## zsh
``` sh
ZSH_THEME="powerlevel9k/powerlevel9k"

plugins=(git
    autojump
    zsh-autosuggestions
    zsh-syntax-highlighting
)

POWERLEVEL9K_MODE='nerdfont-complete'
export TERM="xterm-256color"
POWERLEVEL9K_CUSTOM_WIFI_SIGNAL="zsh_wifi_signal"
POWERLEVEL9K_CUSTOM_WIFI_SIGNAL_BACKGROUND="white"
POWERLEVEL9K_CUSTOM_WIFI_SIGNAL_FOREGROUND="black"

zsh_wifi_signal(){
        local output=$(/System/Library/PrivateFrameworks/Apple80211.framework/Versions/A/Resources/airport -I)
        local airport=$(echo $output | grep 'AirPort' | awk -F': ' '{print $2}')

        if [ "$airport" = "Off" ]; then
                local color='%F{black}'
                echo -n "%{$color%}Wifi Off"
        else
                local ssid=$(echo $output | grep ' SSID' | awk -F': ' '{print $2}')
                local speed=$(echo $output | grep 'lastTxRate' | awk -F': ' '{print $2}')
                local color='%F{black}'

                [[ $speed -gt 100 ]] && color='%F{black}'
                [[ $speed -lt 50 ]] && color='%F{red}'

                echo -n "%{$color%}$speed Mbps \uf1eb%{%f%}" # removed char not in my PowerLine font
        fi
}

POWERLEVEL9K_CONTEXT_TEMPLATE='%n'
POWERLEVEL9K_CONTEXT_DEFAULT_FOREGROUND='white'
POWERLEVEL9K_BATTERY_CHARGING='yellow'
POWERLEVEL9K_BATTERY_CHARGED='green'
POWERLEVEL9K_BATTERY_DISCONNECTED='$DEFAULT_COLOR'
POWERLEVEL9K_BATTERY_LOW_THRESHOLD='10'
POWERLEVEL9K_BATTERY_LOW_COLOR='red'
POWERLEVEL9K_MULTILINE_FIRST_PROMPT_PREFIX=''
POWERLEVEL9K_BATTERY_ICON='\uf1e6 '
POWERLEVEL9K_PROMPT_ON_NEWLINE=true
POWERLEVEL9K_MULTILINE_LAST_PROMPT_PREFIX="%F{014}\u2570%F{cyan}\uF460%F{073}\uF460%F{109}\uF460%f "
POWERLEVEL9K_VCS_MODIFIED_BACKGROUND='yellow'
POWERLEVEL9K_VCS_UNTRACKED_BACKGROUND='yellow'
POWERLEVEL9K_VCS_UNTRACKED_ICON='?'

POWERLEVEL9K_LEFT_PROMPT_ELEMENTS=(context dir vcs)
POWERLEVEL9K_RIGHT_PROMPT_ELEMENTS=(status time dir_writable ip ram battery)

POWERLEVEL9K_SHORTEN_DIR_LENGTH=1

POWERLEVEL9K_TIME_FORMAT="%D{\uf017 %H:%M \uf073 %d/%m/%y}"
POWERLEVEL9K_TIME_BACKGROUND='white'
POWERLEVEL9K_RAM_BACKGROUND='yellow'
POWERLEVEL9K_LOAD_CRITICAL_BACKGROUND="white"
POWERLEVEL9K_LOAD_WARNING_BACKGROUND="white"
POWERLEVEL9K_LOAD_NORMAL_BACKGROUND="white"
POWERLEVEL9K_LOAD_CRITICAL_FOREGROUND="red"
POWERLEVEL9K_LOAD_WARNING_FOREGROUND="yellow"
POWERLEVEL9K_LOAD_NORMAL_FOREGROUND="black"
POWERLEVEL9K_LOAD_CRITICAL_VISUAL_IDENTIFIER_COLOR="red"
POWERLEVEL9K_LOAD_WARNING_VISUAL_IDENTIFIER_COLOR="yellow"
POWERLEVEL9K_LOAD_NORMAL_VISUAL_IDENTIFIER_COLOR="green"
POWERLEVEL9K_HOME_ICON=''
POWERLEVEL9K_HOME_SUB_ICON=''
POWERLEVEL9K_FOLDER_ICON=''
POWERLEVEL9K_STATUS_VERBOSE=true
POWERLEVEL9K_STATUS_CROSS=true
```
## IntelliJ IDEA / Android Stuido

* Android Drawable Preview
* CodeGlance2
* Github Copilot
* JSON To Kotlin Class
* Raninbow Brackets
* PlantUML intergration
* Translation
* ADB Wi-Fi
* Android Parcelable Code Generator

## Chrome 插件
* AdBlock
* AdGuard 广告拦截器
* APK Downloader
* Octotree - GitHub code tree
* uBlacklist
* Unsplash Instant
* YouTube™ 双字幕
* Google 翻译

# 软件下载
[MacWK](https://macwk.com/)