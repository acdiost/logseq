- ```bash
  hostnamectl set-hostname cdh001.domain.com
  
  yum install ansible wget lrzsz -y
  
  # ansible 主机指纹确认
  sed -i 's/^#host_key_checking = False/host_key_checking = False/' /etc/ansible/ansible.cfg
  
  # 在 /etc/ansible/hosts 中填写
  # server 节点
  [server]
  10.111.30.8    hostname=cdh001.domain.com ansible_ssh_port=22 ansible_ssh_user=root ansible_ssh_pass=123456
  
  # agent 节点
  [agent]
  10.111.30.9    hostname=cdh002.domain.com ansible_ssh_port=22 ansible_ssh_user=root ansible_ssh_pass=123456
  10.111.30.10   hostname=cdh003.domain.com ansible_ssh_port=22 ansible_ssh_user=root ansible_ssh_pass=123456
  
  
  # 编辑 hosts 文件
  vi /etc/hosts
  ansible agent -m copy -a 'src=/etc/hosts dest=/etc/hosts'
  
  
  # 关闭防火墙
  ansible all -m service -a 'name=firewalld state=stopped enabled=no'
  
  # selinux
  ansible all -m shell -a "setenforce 0"
  ansible all -m shell -a 'sed -i "s/^SELINUX=.*$/SELINUX=disabled/" /etc/selinux/config'
  
  
  # 时间
  ansible all -m yum -a 'name=chrony state=latest'
  ansible all -m service -a 'name=chronyd state=started enabled=yes'
  ansible all -m shell -a "timedatectl set-timezone Asia/Shanghai"
  ansible all -m shell -a "chronyc sources"
  
  
  # java
  ansible all -m yum 'name=java-1.8 state=latest'
  
  
  # mysql
  yum install maraidb-server mysql
  mysql_secrue_installation
  
  create database hive amon hue oozie
  
  create user 'hive'@'%' identified by 'hive';
  
  
  
  ```