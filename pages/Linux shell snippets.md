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
  ```