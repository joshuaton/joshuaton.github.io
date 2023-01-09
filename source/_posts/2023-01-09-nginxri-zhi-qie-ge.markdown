---
layout: post
title: "nginx日志切割"
date: 2023-01-09 14:56:40 +0800
comments: true
categories: web
---

随着nginx日志越来越大，最终会将服务器上的日志占满，方案是用logformat自动进行日志切割，需要以下步骤

<!--more-->

### 创建logformat配置文件

```
/data/logs/*.log {
  daily
  missingok
  rotate 7
  ifempty
  create 640 nginx root
  sharedscripts
  dateext
  postrotate
    if [ -f /var/run/nginx.pid ]; then
      kill -USR1 \`cat /var/run/nginx.pid\`
    fi
  endscript
}
```

daily表示日志按天切割

rotate 7 表示只保留最近的7条切割文件

postrotate 表示切割日志后要运行的命令，这里是杀掉nginx进程，使其能重新用新文件写入日志

### 修改Dockerfile

```
FROM xxx/nginx:1.18

# 安装logrotate
RUN apt-get update && apt-get -y install logrotate

# 设置时区
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo '$TZ' > /etc/timezone

COPY ${domain}/. /usr/share/nginx/html/
COPY default.conf /etc/nginx/conf.d/default.conf
COPY logrotate-nginx /etc/logrotate.d/
WORKDIR /usr/share/nginx/html
EXPOSE 80

CMD service cron start && nginx -g 'daemon off;'
```

主要增加了以下内容

1. 安装logrotate，因为原始镜像里没有
2. 设置时区，这样日志的时间是准确的，不会早8个小时
3. 将logrotate的配置文件copy到配置目录下
4. 启动crontab，并且修改nginx的启动方式

### 容器运行之后的效果

查看容器的crontab内容

```
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
```

可以看到每天6点25分会运行/etc/cron.daily

/etc/cron.daily/logrotate 是一个脚本，每天会定时调用，执行logrotate二进制命令

```
#!/bin/sh

# skip in favour of systemd timer
if [ -d /run/systemd/system ]; then
    exit 0
fi

# this cronjob persists removals (but not purges)
if [ ! -x /usr/sbin/logrotate ]; then
    exit 0
fi

/usr/sbin/logrotate /etc/logrotate.conf
EXITVALUE=$?
if [ $EXITVALUE != 0 ]; then
    /usr/bin/logger -t logrotate "ALERT exited abnormally with [$EXITVALUE]"
fi
exit $EXITVALUE
```

### 切割后的日志

```
-rw-r----- 1 nginx root  1241741 Sep  9 15:11 nginx_error_log.log-20220909
-rw-r----- 1 nginx root   639982 Sep 13 06:04 nginx_error_log.log-20220912
-rw-r----- 1 nginx root 26065270 Sep 13 19:40 nginx_error_log.log-20220910
-rw-r----- 1 nginx root  5055874 Sep 14 06:38 nginx_error_log.log-20220911
-rw-r----- 1 nginx root   312747 Sep 14 12:17 nginx_error_log.log-20220913
-rw-r----- 1 nginx root  1343155 Sep 15 06:39 nginx_error_log.log-20220914
-rw-r----- 1 nginx root  2816323 Sep 15 14:35 nginx_error_log.log-20220915
-rw-r----- 1 nginx root  1141083 Sep 15 20:13 nginx_error_log.log
```
