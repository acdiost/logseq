- ## 用到的工具
	- ● cowsay
	  ● figlet
	- ● toilet
	  ● fortune
- ## 修改的文件
- ● /etc/issue 输入账户前展示，只会在普通登录时才会显示，ssh 不展示
  ● /etc/ssh/banner 输入密码前展示
  ● /etc/motd 登录成功后展示
  ● /etc/profile.d/  自定义脚本
- > /etc/issue:
  \\S
  Kernel \r on an \m
  \d 显示当前日期；
  \l 显示虚拟控制台号；
  \m 显示机器类型，即 CPU 架构，如 i386 或 x86_64 等（相当于 uname -m）；
  \n 显示主机的网络名（相当于 uname -n）；
  \o 显示域名；
  \r 显示 Kernel 内核版本号（相当于 uname -r）；
  \t 显示当前时间；
  \s 显示当前操作系统名称；
  \u 显示当前登录用户的编号，\U 显示当前登录用户的编号和用户；
  \v 显示当前操作系统的版本日期；
- ## 脚本
	- ```bash
	  #/bin/bash
	  # author: acdiost
	  yum install -y figlet cowsay sysstat
	  
	  figlet -c Welcome | tee >> /etc/issue
	  figlet -c Welcome | tee /etc/ssh/banner
	  figlet -ck acdiost | tee /etc/motd
	  
	  sed -i 's@#Banner none@Banner /etc/ssh/banner@' /etc/ssh/sshd_config
	  systemctl restart sshd
	  
	  cat <<EOF | sudo tee /etc/profile.d/acdiost.sh
	  # 进入系统前检查基本信息
	  # author: acdiost
	  
	  export HISTTIMEFORMAT="%F %T \$(who -u am i 2>/dev/null | awk '{print \$NF}' | sed -e 's/[()]//g') \$(whoami) "
	  
	  banner_file=\$(mktemp)
	  
	  uname -a > \${banner_file}
	  echo "=================================== 系统状态 ===================================" >> \${banner_file}
	  uptime >> \${banner_file}
	  echo "=================================== 内存状态 ===================================" >> \${banner_file}
	  free -h >> \${banner_file}
	  echo "=================================== 磁盘状态 ===================================" >> \${banner_file}
	  df -h | awk 'BEGIN {FS="[ %]+"} {if(\$5 >= 70) {print \$0}}' >> \${banner_file}
	  echo "=================================== IO  状态 ===================================" >> \${banner_file}
	  iostat -cdx >> \${banner_file}
	  
	  cat \${banner_file} | cowsay -n -f \$(cowsay -l | grep -v files | tr " " "\n" | shuf -n 1)
	  rm -f \${banner_file}
	  EOF
	  
	  chmod +x /etc/profile.d/acdiost.sh
	  ```
- ## figlet
- 字体库：http://www.figlet.org/fontdb.cgi
  下载到该目录下：/usr/share/figlet/
- ## cowsay
- 修改展示文件 /usr/share/cowsay/
- ```
  # 随机输出
  cowsay -f `cowsay -l | grep -v files | tr " " "\n" | shuf -n 1`
  
  # 查看所有 cowsay
  cowsay -l | grep -v files | sed "s/ /\n/g" | xargs -i cowsay -f {} {}
  
  # Ubuntu 彩色输出
  toilet -f standard -F gay acdiost
  ```
- ## ansible 关闭 cowsay
- 在 ansible.cnf 中配置 nocows = 1