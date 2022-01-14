- ## 系统日志
- 修改日志保存时间以满足等保要求
- CentOS `/etc/logrotate.conf`
  ```conf
  # 保存一年
  monthly
  rotate 12
  create
  dateext
  include /etc/logrotate.d
  # 存放用户登录的信息
  /var/log/wtmp {
    monthly
    create 0644 root utmp
        minsize 1M
    rotate 6
  }
  # 存放用户登录失败的信息
  /var/log/btmp {
    missingok
    monthly
    create 0600 root utmp
    rotate 6
  }
  ```
- Ubuntu /etc/logrotate.d/rsyslog
  ```conf
  /var/log/syslog
  {
  	rotate 24
  	weekly
  	missingok
  	notifempty
  	delaycompress
  	compress
  	postrotate
  		/usr/lib/rsyslog/rsyslog-rotate
  	endscript
  }
  
  /var/log/mail.info
  /var/log/mail.warn
  /var/log/mail.err
  /var/log/mail.log
  /var/log/daemon.log
  /var/log/kern.log
  /var/log/auth.log
  /var/log/user.log
  /var/log/lpr.log
  /var/log/cron.log
  /var/log/debug
  /var/log/messages
  {
  	rotate 24
  	weekly
  	missingok
  	notifempty
  	compress
  	delaycompress
  	sharedscripts
  	postrotate
  		/usr/lib/rsyslog/rsyslog-rotate
  	endscript
  }
  ```
- 完成后重启 `systemctl restart rsyslog`