title: "Debian 8安装可能会出现的问题"
date: 2015-12-17 15:29:17
tags:
- debian
- linux
---

## apt-get提示磁盘插入
apt-get时会出现类似以下错误
`Media change: please insert the disc labeled
 'Debian GNU/Linux 7.3.0 _Wheezy_ - Official i386 DVD Binary-1 20131215-03:40'
in the drive '/media/cdrom/' and press enter`

解决方案：
修改文件 `sudo nano /etc/apt/sources.list`，将 `deb cdrom:`这一行注释掉，如
```
# deb cdrom:[Debian GNU/Linux 7.0.0 _Wheezy_ - Official amd64 CD Binary-1 20130$

# deb cdrom:[Debian GNU/Linux 7.0.0 _Wheezy_ - Official amd64 CD Binary-1 20130$

deb http://ftp.us.debian.org/debian/ wheezy main
deb-src http://ftp.us.debian.org/debian/ wheezy main

deb http://security.debian.org/ wheezy/updates main
deb-src http://security.debian.org/ wheezy/updates main

# wheezy-updates, previously known as 'volatile'
deb http://ftp.us.debian.org/debian/ wheezy-updates main
deb-src http://ftp.us.debian.org/debian/ wheezy-updates main
```

## 没有安装sudo
1. 打开终端
2. 切换到root用户
    - 执行 `su` 命令
    - 输入密码
3. 安装sudo 
    - `apt-get install sudo`
4. 将用户添加到sudo group
    - `adduser yourusername sudo`
5. 将用户名添加到 **/etc/sudoers** 文件
    - 执行 `nano /etc/sudoers`
    - 找到 `%sudo ALL=(ALL:ALL) ALL` 这一行
    - 在下面插入 `yourusername ALL=(ALL:ALL) ALL` (这句话好奇怪)
    - 保存
6. OK了
