---
layout: post
title: "Linux logrotate 日志分割"
description: "Linux logrotate 日志分割"
keywords: "logrotate 日志分割"
categories: "linux"
background: '/assets/images/bg-post.jpg'

---

Rails在默认情况下会将系统日志全部写在一个文件里，这样当服务器跑的时间比较久之后，日志文件就会变得比较大，而且出了错误，也不方便排查。

logrotate就是管理这些日志文件的神器，可以对单个日志文件或者某个目录下的文件按时间/大小进行切割，压缩操作；指定日志保存数量；还可以在切割之后运行自定义命令。

##### logrotate 分割日志原理
系统会按照计划的频率运行logrotate，logrotate需要的cron任务应该在安装时就自动创建了。可以使用以下命令查看

```
cat /etc/cron.daily/logrotate
```

##### logrotate.conf 主配置文件
logrotate的主要配置文件位于 /etc/logrotate.conf ，在这个文件中有一行需要特别注意一下, 这个目录就是各软件使用logrotate分割日志文件所使用的配置文件


```
include /etc/logrotate.d
```

##### logrotate.d 配置文件目录
例如：

```bash
drwxr-xr-x  2 root root 4096 Mar 10 09:52 ./
drwxr-xr-x 96 root root 4096 Jan 21 11:02 ../
-rw-r--r--  1 root root  174 Mar 10 09:52 724activity
-rw-r--r--  1 root root  173 Apr 10  2014 apt
-rw-r--r--  1 root root   79 Feb 18  2014 aptitude
-rw-r--r--  1 root root  232 Mar  7  2014 dpkg
-rw-r--r--  1 root root  847 Oct 25 04:08 mysql-server
-rw-r--r--  1 root root   94 Jan 23  2013 ppp
-rw-r--r--  1 root root  126 Jan 15  2014 redis-server
-rw-r--r--  1 root root  515 Dec  4  2013 rsyslog
-rw-r--r--  1 root root  178 Mar  1  2014 ufw
-rw-r--r--  1 root root  122 Apr 12  2014 upstart
```

##### 配置自定义需要切割的文件
直接去 /etc/logrotate.d 下随便复制一个文件，改名，然后直接进去看一下当中的英文，改一下内容即可

```bash
/var/www/ruby-china/current/log/*.log {
  daily
  missingok
  rotate 7
  compress
  dateext
  delaycompress
  lastaction
    pid=/data/www/ruby-china/current/tmp/pids/unicorn.pid
    sudo test -s $pid && sudo kill -USR1 "$(cat $pid)"
  endscript
}
```

设定好之后，可以等明天，或是执行
`logrotate -f /etc/logrotate.conf` 看看。

##### logrotate 参数说明

```
1. daily 表示每天整理，也可以改成 weekly 或 monthly
2. dateext 表示檔案補上 rotate 的日期
3. missingok 表示如果找不到 log 檔也沒關係
4. rotate 7 表示保留7份
5. compress 表示壓縮起來，預設用 gzip。不過如果硬碟空間多，不壓也沒關係。
6. delaycompress 表示延後壓縮直到下一次 rotate
7. notifempty 表示如果 log 檔是空的，就不 rotate
8. copytruncate 先複製 log 檔的內容後，在清空的作法，因為有些程式一定 log 在本來的檔名，例如 rails。另一種方法是 create。
```