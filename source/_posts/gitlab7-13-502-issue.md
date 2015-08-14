title: gitlab 7.13 502 issue
date: 2015-08-14 15:58:25
tags:
- gitlab
---

单核服务器在gitlab重启之后常常会出现502错误，是因为”unicorn”启动比较慢，造成延迟引起的。我们可以手动执行以下命令解决:
```shell
sudo gitlab-ctl hup unicorn
sudo gitlab-ctl restart sidekiq
```
