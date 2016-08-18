title: Linux命令行利器Tmux
tags:
  - linux
  - tmux
categories: []
date: 2016-06-04 15:36:00
---
## 简介
Tmux是一个优秀的终端复用软件，即使非正常掉线，也能保证当前的任务运行，这一点对于 远程SSH访问特别有用，网络不好的情况下仍然能保证工作现场不丢失!此外，tmux完全使用键盘 控制窗口，实现窗口的切换功能。

## 安装
```
sudo apt-get install tmux
```
安装完成后用“tmux”命令启动。

## 配置
在~/下建立".tmuix.conf"文件
```
set -g default-terminal "screen-256color"

# -- base -- #
unbind C-b
set -g prefix C-q
set -g status-keys vi
setw -g mode-keys vi
bind : command-prompt
bind r source-file ~/.tmux.conf \; display-message "Reloading..".
set -g default-terminal "screen-256color"
bind-key a send-prefix

# -- windown -- #
bind s split-window -h -c "#{pane_current_path}"
bind v split-window -v -c "#{pane_current_path}"
bind-key c  new-window -c "#{pane_current_path}"

bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

bind ^k resizep -U 10
bind ^j resizep -D 10
bind ^h resizep -L 10
bind ^l resizep -R 10
bind ^u swapp -U
bind ^d swapp -D

bind u choose-session
bind o choose-window
bind \ last
bind q killp

bind-key -n C-S-Left swap-window -t -1
bind-key -n C-S-Right swap-window -t +1
set -g base-index 1
setw -g pane-base-index 1
set -g history-limit 5000

# pane border
set -g pane-border-fg black
set -g pane-border-bg white
set -g pane-active-border-fg black
set -g pane-active-border-bg '#afd787'

# -- command -- #
bind m command-prompt "splitw 'exec man %%'"
bind space copy-mode
bind -t vi-copy v begin-selection
bind -t vi-copy y copy-selection
bind -t vi-copy C-v rectangle-toggle
bind ] paste-buffer

# -- statusbar --#
set -g status-justify centre
set -g status-right-attr bright
set -g status-right "%H:%M %a %m-%d"
set -g status-bg default
set -g status-fg '#afd787'
setw -g window-status-current-attr bright
setw -g window-status-current-fg black
setw -g window-status-current-bg '#afd787'
set -g status-utf8 on
set -g status-interval 1

# -- mouse --#
setw -g mouse-resize-pane on 
setw -g mouse-select-pane on 
setw -g mouse-select-window on 
setw -g mode-mouse on
```

## 快捷键
按照上述配置，常用快捷键位
ctrl-q s 横向分割窗口
ctrl-q v 纵向分割窗口
ctrl-q x 关闭当前窗口
ctrl-q h,j,k,l  切换窗口
ctrl-q ctrl-h,j,k,l 调整窗口尺寸

## 最终效果图
![tmux](/img/tmux-screen.png)