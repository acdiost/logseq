- ```bash
  # 安装 firewalld 防火墙
  yum install firewalld
  
  # 开启服务
  systemctl start firewalld.service
  
  # 关闭防火墙
  systemctl stop firewalld.service
  
  # 开机自动启动
  systemctl enable firewalld.service
  
  # 关闭开机启动
  systemctl disable firewalld.service
  
  firewall-cmd --state 
  # running 表示运行
  
  # 添加服务
  firewall-cmd --permanent --zone=public --add-service=https
  
  # 添加端口
  firewall-cmd --permanent --zone=public --add-port=8080-8081/tcp
  
  # 列出服务
  firewall-cmd --permanent --zone=public --list-services 
  # dhcpv6-client ssh
  
  # 列出端口
  firewall-cmd --permanent --zone=public --list-ports
  
  # 添加 IP
  firewall-cmd --permanent --zone=public --add-rich-rule="rule family="ipv4" source address="192.168.0.1/24" service name="http" accept"
  
  # 移除 IP
  firewall-cmd --permanent --zone=public --remove-rich-rule="rule family="ipv4" source address="192.168.0.1/24" service name="http" accept"
  
  ```