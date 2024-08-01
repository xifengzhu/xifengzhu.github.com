---
layout: post
title: "使用 Let’s Encrypt 加密(HTTPS)你的网站"
description: "使用 Let’s Encrypt 加密(HTTPS)你的网站"
keywords: "Let’s Encrypt nginx https"
categories: "部署"
background: '/assets/images/bg-post.jpg'

---

使用 Let’s Encrypt 加密(HTTPS)你的网站

#### 下载 Let’s Encrypt 客户端

```bash
$ sudo git clone https://github.com/certbot/certbot /opt/letsencrypt
```

#### 允许 Let’s Encrypt 访问文件

修改NGINX配置，允许 Let’s Encrypt 访问文件

1. 将此位置块添加到 HTTP 通信的虚拟服务器

```bash
server {
  listen 80 default_server;
  server_name activity-api.724.org.cn;

  location /.well-known/acme-challenge {
    root /var/www/724activity_staging/current/public;
  }
  ...
}
```
2. 在语法上验证配置文件并且重启 nginx 服务

```
sudo service nginx restart
```

#### 为域名请求证书

```
./certbot-auto certonly -d activity-api.724.org.cn -d activity-wechat.724.org.cn
```

可能出现如下错误：

```bash
Traceback (most recent call last):
  File "/usr/lib/python3/dist-packages/virtualenv.py", line 2363, in <module>
    main()
  File "/usr/lib/python3/dist-packages/virtualenv.py", line 719, in main
    symlink=options.symlink)
  File "/usr/lib/python3/dist-packages/virtualenv.py", line 988, in create_environment
    download=download,
  File "/usr/lib/python3/dist-packages/virtualenv.py", line 918, in install_wheel
    call_subprocess(cmd, show_stdout=False, extra_env=env, stdin=SCRIPT)
  File "/usr/lib/python3/dist-packages/virtualenv.py", line 812, in call_subprocess
    % (cmd_desc, proc.returncode))
OSError: Command /opt/eff.org/certbot/venv/bin/python2.7 - setuptools pkg_resources pip wheel failed with error code 1
```

解决方法：

```bash
$ export LC_ALL="en_US.UTF-8"
$ export LC_CTYPE="en_US.UTF-8"
```

选择将证书安装在  `webroot` 下面

```bash
How would you like to authenticate with the ACME CA?
-------------------------------------------------------------------------------
1: Nginx Web Server plugin - Alpha (nginx)
2: Spin up a temporary webserver (standalone)
3: Place files in webroot directory (webroot)
-------------------------------------------------------------------------------
Select the appropriate number [1-3] then [enter] (press 'c' to cancel): 3
```

输入项目部署的地址, 例如: `/var/www/kdaibiao/current/public`

#### 在 NGINX 中添加证书配置

1. 将证书和密钥添加到 HTTP 通信的服务器块

```bash
server {
  listen 443 ssl default_server;
  server_name activity-api.724.org.cn;

  ssl_certificate /etc/letsencrypt/live/activity-api.724.org.cn/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/activity-api.724.org.cn/privkey.pem;
  ...
}
```

2. 在语法上验证配置文件并且重启 nginx 服务并加载证书

```bash
sudo service nginx restart
```

#### 自动续期

Certbot 可以配置为在到期之前自动续订您的证书。加密证书有效期为 90 天，自动续期功能非常实用。通过运行此命令，您可以为您的证书测试自动续订

```bash
./opt/letsencrypt/certbot-auto renew --dry-run
```

如果上述命令正确工作，可以为您安排自动更新添加一个 cron 或systemd 的任务，运行以下︰

```
//执行
sudo crontab -e
//添加一行
30 4 * * 1 /opt/letsencrypt/letsencrypt-auto renew --renew-hook "service nginx restart" --quiet > /dev/null 2>&1 &
```

#### Refs
* https://certbot.eff.org/#ubuntuother-nginx
* https://www.nginx.com/blog/free-certificates-lets-encrypt-and-nginx/