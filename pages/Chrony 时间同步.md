- ## 安装配置
- ```bash
  timedatectl set-timezone Asia/Shanghai
  
  yum install chrony -y
  
  firewall-cmd --add-service=ntp --permanent --zone=public
  firewall-cmd --reload
  
  systemctl enable chronyd
  systemctl start chronyd
  ```
- ## 配置文件
- `/ect/chrony.conf (ubuntu: /etc/chrony/)`
- ### 客户端
	- id:: 620da90c-4bee-4519-9f81-cc4252e5a929
	  ```
	  server ntp1.aliyun.com iburst
	  server ntp2.aliyun.com iburst
	  server ntp3.aliyun.com iburst
	  driftfile /var/lib/chrony/drift
	  makestep 1.0 3
	  rtcsync
	  keyfile /etc/chrony.keys
	  logdir /var/log/chrony
	  ```
- ### 服务端
	- ```
	  server pool.ntp.org iburst
	  driftfile /var/lib/chrony/drift
	  makestep 1.0 3
	  rtcsync
	  keyfile /etc/chrony.keys
	  allow 192.168.0.0/16
	  local stratum 10
	  logdir /var/log/chrony
	  
	  
	  stratumweight 0
	  #bindcmdaddress 127.0.0.1
	  #bindcmdaddress ::1
	  commandkey 1
	  generatecommandkey
	  noclientlog
	  logchange 0.5
	  ```
- ## 常用命令
	- ```bash
	  chronyc -a makestep
	  chronyc activity
	  chronyc accheck
	  chronyc add server
	  chronyc clients
	  chronyc delete
	  chronyc tracking
	  chronyc sources -v
	  chronyc settime
	  ```