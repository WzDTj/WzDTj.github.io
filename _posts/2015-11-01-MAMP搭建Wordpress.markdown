---
layout: post
title:  "MAMP 搭建 Wordpress"
date:   2015-11-02 03:40:36 +0800
categories: 
---
最近，想将一些杂七杂八的东西，集中在博客上，也方便我写写心得、扯扯淡。

这个博客建了好久，由于拖延症、强迫症各种处女座综合症，最终也就荒废了。还好有备份，以至于不需要重新开始写。

这个博客的起初应该是好几年前建的吧，想想每年只需要维护一小部分费用就好了，但终归我是个省不下钱的人，总是买一年隔两年。最终还是在我的MacBook上乖乖的躺着了。之前图方便一直是通过XAMMP在搭建的MacBook上的。可能是太久没折腾了，也可能是因为强迫症，不喜欢多装应用程序，也不喜欢重复的。毅然决然的将XAMMP卸载，自己从零开始构建这个博客， 虽然遇到了许多让我抓耳挠腮的问题，但终归还是解决了。

思路
---
通过Apache的虚拟主机来构建两个站点，一个博客，一个phpMyAdmin数据库管理站点。

我们需要什么？

 - Mac OS X El Capitan`10.11.2 Beta`
 - HomeBrew`0.9.5`
 - Apache`2.4.16 Unix`
 - php`5.5.30`
 - MySQL`5.6.27 Homebrew`
 - phpMyAdmin`4.5.1`
 - Wordpress`4.3.1`

Apache、php在 OS X El Capitan中已经预装好了。
[Wordpress](https://cn.wordpress.org) 可以在其官网下载。

MySQL和phpMyAdmin可以通过HomeBrew安装。官方网站下载也是可以的。

所以我们从安装HomeBrew开始。

HomeBrew
---

 - 安装Xcode，通过[App Store](https://itunes.apple.com/us/app/xcode/id497799835?ls=1&amp;mt=12)下载安装。
 - 打开Xcode并同意协议、条款。
 - 安装命令行工具  
`install Command Line Tools">xcode-select --install`
 - 安装HomeBrew  
`ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"`
 - 确保HomeBrew安装正确  
`brew --version`

如果在之前的Mac OS X版本中安装过HomeBrew可能回遇到权限问题  
`sudo chown -R $(whoami):admin /usr/local`

没问题的话，HomeBrew就装好了，有问题就Google解决下，也可以通过执行  
`brew doctor`  
来返回一些建议。

安装完成后，我们就可以安装 `MySQL` 和 `phpMyAdmin`

```
brew install mysql
brew install phpmyadmin
```

> Homebrew会将东西安装在 `/usr/local/Cellar`

建几个文件夹
---

为我们的站点先建立好目录，并把博客需要的东西放进去。
我们要在用户 `Home` 目录下建立一个 `Sites` 文件夹，表明我们站点的目录。并把我们需要的 `Wordpress` 和 `phpmyadmin`复制进去。

```
mkdir ~/Sites
cd ~/Sites
cp -R ~/Downloads/wordpress/ ~/Sites/blog/
cp -R /usr/local/Cellar/phpmyadmin/'version'/share/phpmyadmin/ ~/Sites/phpMyAdmin/
```

很好，我们准备工作做好了，开始搭积木吧。

Apache
---
先来了解Apache的几个命令：

```
sudo apachectl start  #启动apache
sudo apachectl stop  #关闭apache
sudo apachectl -k restart  #立即重启apache
sudo apachectl -t  #检查配置文件中是否有错误
```

 - 开始配置 `Apache`

`sudo vim /etc/apache2/httpd.conf`

 - 开启虚拟主机

`Include /private/etc/apache2/extra/httpd-vhosts.conf #去掉前面的‘＃’注释`

 - 设置网站根目录

```
DocumentRoot "/Users/[your_user]/Sites"  #就是我们前面建立好的站点目录。
<Directory "/Users/[your_user]/Sites">
    AllowOverride None
    Allow from all
</Directory>
```

 - 配置虚拟主机

`sudo vim /etc/apache2/extra/httpd-vhosts.conf`

将原来的的<Virtualhost>块注释掉,并添加自己的虚拟主机.

```
<VirtualHost *:80>
    DocumentRoot "/Users/WzDTj/Sites/blog"
    ServerName blog
</VirtualHost>

<VIrtualHost *:88>
    DocumentRoot "/Users/WzDTj/Sites/phpmyadmin"
    ServerName phpmyadmin
</VirtualHost>
```

 - 启动 `Apache`  
`sudo apachectl start`

PHP
---

 - 配置 `/etc/php.ini`  
`cp /etc/php.ini.default /etc/php.ini`

MySQL
---

 - 启动MySQL  
`mysql.server start`

 - 设置root用户的密码  
`mysqladmin -uroot password 'your_password'`

phpMyAdmin
---

 - 配置 `php.ini`

```
cp ~/Sites/phpmyadmin/config.simple.inc.php ~/Sites/phpmyadmin/config.inc.php 
#修改下面字段的值为127.0.0.1。
$cfg['Servers'][$i]['host'] = '127.0.0.1'
```

 - 通过 `phpMyAdmin` 来管理 `MySQL`  
`http://localhost:88`

Wordpress
---

- 访问 `http://localhost`
跟着指导一步一步来，你就可能完成博客的搭建了。

一些问题的解决方法
---

Wordpress无法进入后台，不断转跳登录页面。 

> 我当时遇到这个问题是因为新建了一个数据库用户来管理博客的数据库。但是没有给予全局的update权限导致的。
赋予这个用户update权限。

无法启动MySQL，提示Error2002。

> 莫名其妙，我也忘了怎么回事，反正就是mysql.sock丢失了。

> 通过mysqladmin -h 127.0.0.1 -P 3306 -u root -p shutdown来关闭。  
> 最好在my.cnf中socket字段中指明mysql.sock的位置，因为每次mysql启动的时候，就会自动建立mysql.sock。  
> 还有就是在php.ini中把do_mysql.default_socket、mysql.default_socket、mysql.default_socket字段中指明mysql.sock的位置，默认会在/tmp中寻找mysql.sock。

参考
---
[GRAV](http://getgrav.org/blog/mac-os-x-apache-setup-multiple-php-versions)
