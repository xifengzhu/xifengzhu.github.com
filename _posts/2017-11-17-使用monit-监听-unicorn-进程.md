---
layout: post
title: 使用Monit 监听 Unicorn 进程
background: '/assets/images/bg-post.jpg'
---

 Monit 可以自动监视和管理服务器程序, 以确保它们不仅始终保持联机状态, 而且文件大小、校验和或权限始终正确。此外, 播还提供了一个基本的 web 界面, 通过它可以设置所有进程。

### 安装 Monit

安装 `monit`
```shell
sudo apt-get install monit
```

通过命令 `sudo monit` 启动 `monit`

查看下状态 `monit status`

```shell
The Monit daemon 5.3.2 uptime: 1h 25m

System 'myhost.mydomain.tld'
  status                            Running
  monitoring status                 Monitored
  load average                      [0.03] [0.14] [0.20]
  cpu                               3.5%us 5.9%sy 0.0%wa
  memory usage                      26100 kB [10.4%]
  swap usage                        0 kB [0.0%]
  data collected                    Thu, 30 Aug 2012 18:35:00
```

修改配置文件 `/etc/monit/monitrc`，打开web界面访问

```shell
set httpd port 2812 and
  use address ${IP} # only accept connection from localhost
  allow localhost   # allow localhost to connect to the server and
  allow 0.0.0.0/0.0.0.0
  allow user:password read-only # require user 'user' with password 'password'
```

配置完成之后重新加载一下 `monit reload`

### 添加监控进程

在 `/etc/monit/conf-available` 目录下添加一个文件例如： `www.example.com`

内容如下:

```
check process example_unicorn
  with pidfile /var/www/www.example.com/current/tmp/pids/unicorn.pid
  start program = "/etc/init.d/example_unicorn start" as uid root and gid root
  stop program = "/etc/init.d/example_unicorn stop" as uid root and gid root
```

如何制作 `example_unicorn` service 请见 [Unicorn auto start after server boot](/blog/2017/08/26/unicorn-auto-start-after-server-boot.html)

并且新建一个软连接：

```shell
ln -s /etc/monit/conf-available/www.example.com /etc/monit/conf-enabled/www.example.com
```

检查下语法 `monit t`, 重新加载下 `monit reload` 即可。

访问 `http://${ip}:2812 `界面截图如下：

![image](https://user-images.githubusercontent.com/4188624/32927828-7bca224c-cb14-11e7-8519-243bd1abddc5.png)

### 参考
* [Monit 官网](https://mmonit.com/monit/)
* [How To Install and Configure Monit](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-monit)
* [Unicorn auto start after server boot](http://xifengzhu.github.io/blog/2017/08/26/unicorn-auto-start-after-server-boot.html)