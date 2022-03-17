- shell 代码片段
- ```bash
  # 禁用 CTRL+C
  #!/usr/bin/env bash
  # 禁用信号,写在脚本开始
  trap "" HUP INT QUIT TSTP
  sleep 5
  
  
  # 磁盘已用存储
  ansible all -m shell -a "hostname -i && hostname -I && df -h | grep -v '文件系统' | grep -v 'M' | awk '{print \$3}' | grep G | cut -dG -f1 | awk '{sum += \$1} END {print sum}'"
  
  
  # 内存使用率
  total=$(free -m | awk 'BEGIN {FS="[ ]+"} /Mem/{print $2}')
  used=$(free -m | awk 'BEGIN {FS="[ ]+"} /Mem/{print $3}')
  per=$(awk -v x="$total" -v y="$used" 'BEGIN {printf "%.2f\n",y / x * 100}')%
  echo $per
  
  
  # 统计 TCP/IP 建立的连接状态
  netstat -n | awk '/^tcp/{++state[$NF]}END{for(key in state) print key, "\t", state[key]}'
  
  # 清空文件里内容不删除文件
  for i in `find path -type f -size +1G`;do echo > $i;done
  
  
  # 安装最新版本内核
  rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
  yum install https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm
  # 查看
  # yum --disablerepo=\* --enablerepo=elrepo-kernel list available
  yum --enablerepo=elrepo-kernel install kernel-ml
  # 查看系统上的所有可用内核
  awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
  # 设置新的内核
  grub2-set-default 0
  grub2-mkconfig -o /boot/grub2/grub.cfg
  
  
  # bash 信息带颜色提示
  black() {
      echo -e "\e[30m${1}\e[0m"
  }
  
  red() {
      echo -e "\e[31m${1}\e[0m"
  }
  
  green() {
      echo -e "\e[32m${1}\e[0m"
  }
  
  yellow() {
      echo -e "\e[33m${1}\e[0m"
  }
  
  blue() {
      echo -e "\e[34m${1}\e[0m"
  }
  
  magenta() {
      echo -e "\e[35m${1}\e[0m"
  }
  
  cyan() {
      echo -e "\e[36m${1}\e[0m"
  }
  
  gray() {
      echo -e "\e[90m${1}\e[0m"
  }
  
  black 'BLACK'
  red 'RED'
  green 'GREEN'
  yellow 'YELLOW'
  blue 'BLUE'
  magenta 'MAGENTA'
  cyan 'CYAN'
  gray 'GRAY'
  
  
  # 从 FTP 服务器下载文件
  #!/bin/bash
  if [ $# -ne 1 ]; then
      echo "Usage: $0 filename"
  fi
  dir=$(dirname $1)
  file=$(basename $1)
  ftp -n -v << EOF   # -n 自动登录
  open 192.168.1.10  # ftp 服务器
  user admin password
  binary   # 设置 ftp 传输模式为二进制，避免 MD5 值不同或 .tar.gz 压缩包格式错误
  cd $dir
  get "$file"
  EOF
  
  
  # 输入数字运行相应命令
  #!/bin/bash
  echo "*cmd menu* 1-date 2-ls 3-who 4-pwd 0-exit "
  while :
  do
  # 捕获用户键入值
   read -p "please input number :" n
   n1=`echo $n|sed s'/[0-9]//'g`
  # 空输入检测 
   if [ -z "$n" ]
   then
   continue
   fi
  # 非数字输入检测 
   if [ -n "$n1" ]
   then
   exit 0
   fi
   break
  done
  case $n in
   1)
   date
   ;;
   2)
   ls
   ;;
   3)
   who
   ;;
   4)
   pwd
   ;;
   0)
   break
   ;;
      # 输入数字非 1-4 的提示
   *)
   echo "please input number is [1-4]"
  esac
  
  
  # Expect 实现 SSH 免交互执行命令,需安装 yum install -y expect
  USER=root
  PASS=password
  IP=192.168.1.10
  # EOF 输入
  expect << EOF
  set timeout 30
  spawn ssh $USER@$IP
  expect {
  "(yes/no)" {send "yes\r"; exp_continue}
  "password:" {send "$PASS\r"}
  }
  expect "$USER@*"  {send "$1\r"}
  expect "$USER@*"  {send "exit\r"}
  expect eof
  EOF
  # 字符输入
  expect -c "
      spawn ssh $USER@$IP
      expect {
          \"(yes/no)\" {send \"yes\r\"; exp_continue}
          \"password:\" {send \"$PASS\r\"; exp_continue}
          \"$USER@*\" {send \"df -h\r exit\r\"; exp_continue}
      }"
  
  
  # 批量修改服务器用户密码
  # cat old_pass.txt 
  192.168.1.2   root   12345678   22
  192.168.1.3   root   12345678   22
  
  #!/bin/bash
  OLD_INFO=old_pass.txt
  NEW_INFO=new_pass.txt
  for IP in $(awk '/^[^#]/{print $1}' $OLD_INFO); do
      USER=$(awk -v I=$IP 'I==$1{print $2}' $OLD_INFO)
      PASS=$(awk -v I=$IP 'I==$1{print $3}' $OLD_INFO)
      PORT=$(awk -v I=$IP 'I==$1{print $4}' $OLD_INFO)
      NEW_PASS=$(mkpasswd -l 8)  # 随机密码 yum install -y expect
      echo "$IP   $USER   $NEW_PASS   $PORT" >> $NEW_INFO
      expect -c "
      spawn ssh -p$PORT $USER@$IP
      set timeout 2
      expect {
          \"(yes/no)\" {send \"yes\r\";exp_continue}
          \"password:\" {send \"$PASS\r\";exp_continue}
          \"$USER@*\" {send \"echo \'$NEW_PASS\' |passwd --stdin $USER\r exit\r\";exp_continue}
      }"
  done
  
  # cat new_pass.txt 
  192.168.1.2   root   K38pg@jL   22
  192.168.1.3   root   u4Hs5,vJ   22
  
  
  # iptables 自动屏蔽访问网站频繁的 IP
  # 根据访问日志（Nginx为例）
  #!/bin/bash
  DATE=$(date +%d/%b/%Y:%H:%M)
  # 先 tail 防止文件过大，读取慢，数字可调整每分钟最大的访问量。awk 不能直接过滤日志，因为包含特殊字符。
  ABNORMAL_IP=$(tail -n 5000 access.log | grep $DATE | awk '{a[$1]++}END{for(i in a)if(a[i]>100)print i}')
  for IP in $ABNORMAL_IP; do
      if [ $(iptables -vnL | grep -c "$IP") -eq 0 ]; then
          iptables -I INPUT -s $IP -j DROP fidone
      fi
  done
  
  # 通过 TCP 建立的连接
  #!/bin/bash
  # gsub 是将第五列（客户端 IP）的冒号和端口去掉
  ABNORMAL_IP=$(netstat -an | awk '$4~/:80$/ && $6~/ESTABLISHED/{gsub(/:[0-9]+/,"",$5);{a[$5]++}}END{for(i in a)if(a[i]>100)print i}')
  for IP in $ABNORMAL_IP; do
      if [ $(iptables -vnL | grep -c "$IP") -eq 0 ]; then
          iptables -I INPUT -s $IP -j DROP    
      fi
  done
  
  
  # 判断输入的是否为 IP 地址
  #!/bin/bash
  function check_ip(){
      local IP=$1
      VALID_CHECK=$(echo $IP | awk -F. '$1<=255&&$2<=255&&$3<=255&&$4<=255{print "yes"}')
      if echo $IP | grep -E "^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$" >/dev/null; then
          if [ $VALID_CHECK == "yes" ]; then
              return 0
          else
              echo "$IP not available!"
              return 1
          fi
      else
          echo "Format error! Please input again."
          return 1
      fi
  }
  while true; do
      read -p "Please enter IP: " IP
      check_ip $IP
      [ $? -eq 0 ] && break || continue
  done
  ```