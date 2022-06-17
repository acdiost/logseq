- > Author: Dawn
  Date: 2022-04-29
  Version: 1.0.0
  Desc: ambari 部署手册
-
- 安装手册： [https://docs.cloudera.com/HDPDocuments/Ambari-2.7.5.0/bk_ambari-installation/bk_ambari-installation.pdf](https://docs.cloudera.com/HDPDocuments/Ambari-2.7.5.0/bk_ambari-installation/bk_ambari-installation.pdf)
-
- rpm 安装包：https://www.makeopensourcegreatagain.com/rpms/
-
- 安装要求：
	- CentOS 7.9
	- scp, curl, unzip, tar, wget, and gcc* (ansible chrony nginx mysql5.7)
	- OpenSSL
	- Python2.7.12（python-devel*）
	- 至少 1GB 内存，500M 空闲（8G 以上）
	- 20GB 存储
	- 文件描述符大于 10000 `ulimit -Sn && ulimit -Hn`
		- 设置： `ulimit -n 10000`
-
- 环境准备：
	- 主机名设置
	- ssh 免密访问
	- 服务账户设置 (ambari 安装过程中自动创建)
	- 时间同步
	- 防火墙配置
	- selinux 关闭，umask 设置
	- 安装 Java
	- 配置数据库和连接驱动
	- 配置安装源
	- 安装 ambari-server
-
- ---
-
- 配置本地源 （需要提前准备 ambari rpm 离线包）有互联网跳过此步骤，后面有在线源
	- ```bash
	  # 在有互联网的环境准备 rpm 离线包,完成后传输至内网
	  yum localinstall \
	  https://dev.mysql.com/get/mysql57-community-release-el7-8.noarch.rpm
	  
	  yum install --downloadonly --downloaddir=./rpms yum-utils createrepo nginx \
	  ansible unzip gcc* chrony mysql-community-server lrzsz mysql-connector-java* \
	  postgresql-server postgresql
	  
	  tar -czf rpms.tar.gz rpms
	  
	  # 无互联网需本地安装 rpm -i nginx-xxx.rpm createrepo-xxx.rpm
	  yum install createrepo nginx -y
	  
	  rm -rf /usr/share/nginx/html/*
	  
	  # 解压 ambari rpm 包用于制作本地源
	  tar xaf rpms.tar.gz -C /usr/share/nginx/html/rpms
	  tar xaf ambari-2.7.4.0-centos7.tar.gz -C /usr/share/nginx/html/
	  tar xaf HDP-3.1.4.0-centos7-rpm.tar.gz -C /usr/share/nginx/html/
	  tar xaf HDP-GPL-3.1.4.0-centos7-gpl.tar.gz -C /usr/share/nginx/html/
	  tar xaf HDP-UTILS-1.1.0.22-centos7.tar.gz -C /usr/share/nginx/html/
	  
	  
	  # 创建 yum 元数据库
	  cd /usr/share/nginx/html/rpms && createrepo .
	  
	  # 修改 /etc/nginx/nginx.conf 在 server 段落中新增 autoindex on;
	  systemctl enable nginx --now
	  
	  
	  # 基础扩展 rpm 包的源, baseurl 需匹配
	  cat <<EOF | tee /etc/yum.repos.d/base.repo
	  [base]
	  gpgcheck=0
	  enabled=1
	  baseurl=http://ip:port/rpms
	  name=base
	  EOF
	  
	  # 需修改，版本要匹配
	  # 在 ambari web 配置 yum 本地仓库时会自动创建 hdp 的 repo
	  cat <<EOF | tee /etc/yum.repos.d/ambari.repo
	  [ambari]
	  gpgcheck=0
	  enabled=1
	  baseurl=http://ip:port/ambari/centos7/xxx
	  name=Ambari, ambari-2.7.4.0
	  EOF
	  ```
	-
	- > Ambari Base URL            http://<web.server>/Ambari-2.7.5.0/<OS>
	  HDF Base URL(若有)        http://<web.server>/hdf/HDF/<OS>/3.x/updates/<latest.version>
	  HDP Base URL                 http://<web.server>/hdp/HDP/<OS>/3.x/updates/<latest.version>
	  HDP-UTILS Base URL     http://<web.server>/hdp/HDP-UTILS-<version>/repos/<OS>
	-
- 安装基础软件 (无网络需要提前准备离线包 lrzsz chrony ansible nginx wget mysql )
	- ```bash
	  yum install ansible wget lrzsz -y
	  
	  # ansible 主机指纹确认
	  sed -i 's/^#host_key_checking = False/host_key_checking = False/' /etc/ansible/ansible.cfg
	  
	  # 在 /etc/ansible/hosts 中填写
	  # server 节点
	  [server]
	  10.111.30.8    hostname=ambari001.domain.com ansible_ssh_port=22 ansible_ssh_user=root ansible_ssh_pass=123456
	  
	  # agent 节点
	  [agent]
	  10.111.30.9    hostname=ambari002.domain.com ansible_ssh_port=22 ansible_ssh_user=root ansible_ssh_pass=123456
	  10.111.30.10   hostname=ambari003.domain.com ansible_ssh_port=22 ansible_ssh_user=root ansible_ssh_pass=123456
	  
	  
	  # 修改系统编码，中文可能会有问题
	  ansible all -m shell -a "echo 'LANG=en_US.UTF-8' > /etc/locale.conf"
	  
	  # 将基础文件复制至 agent 节点
	  ansible agent -m copy -a 'src=/etc/yum.repos.d/base.repo dest=/etc/yum.repos.d/base.repo'
	  ansible agent -m copy -a 'src=/etc/yum.repos.d/ambari.repo dest=/etc/yum.repos.d/ambari.repo'
	  
	  ```
-
- 设置主机名（填写好 ansible hosts 后可批量设置）
	- 将所有的机器主机名设置好后，写入 `/etc/hosts`
	- ```bash
	  # 满足 FQDN （可选）
	  hostnamectl set-hostname ambari001.domain.com
	  
	  while read LINE
	  do
	      sed -i "/$(echo ${LINE}|grep -oP '(?<=[^a-z])[^ ]*(?= *=)') =/d; $ a${LINE}" /etc/hosts
	  done <<-EOF
	  10.111.30.8    ambari001.domain.com
	  10.111.30.9    ambari002.domain.com
	  10.111.30.10   ambari003.domain.com
	  EOF
	  
	  ansible agent -m copy -a 'src=/etc/hosts dest=/etc/hosts'
	  
	  # ansible-playbook
	  ---
	  - hosts: all
	    remote_user: root
	    gather_facts: true
	  
	    tasks:
	    - name: Set a hostname
	      ansible.builtin.hostname:
	        name: '{{ hostname|quote }}'
	    - name: "Add to hosts"
	      lineinfile:
	        dest: /etc/hosts
	        line: "{{ ansible_default_ipv4.address }}    {{ hostname|quote }}"
	      run_once: True
	  ```
-
- 基础环境优化
	- ```bash
	  vi /etc/rc.local
	  if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
	     echo never > /sys/kernel/mm/transparent_hugepage/enabled
	  fi
	  if test -f /sys/kernel/mm/transparent_hugepage/defrag; then
	     echo never > /sys/kernel/mm/transparent_hugepage/defrag
	  fi
	  echo never > defrag_file_pathname
	  
	  chmod +x /etc/rc.local
	  ansible agent -m copy -a 'src=/etc/rc.local dest=/etc/rc.local'
	  
	  
	  vim /etc/security/limits.conf
	  * soft nofile 65536 
	  * hard nofile 65536 
	  * soft nproc 131072
	  * hard nproc 131072
	  
	  ansible agent -m copy -a 'src=/etc/security/limits.conf dest=/etc/security/limits.conf'
	  
	  
	  vim /etc/systemd/system.conf
	  DefaultLimitNOFILE=1024000
	  DefaultLimitNPROC=1024000
	  
	  ansible agent -m copy -a 'src=/etc/systemd/system.conf dest=/etc/systemd/system.conf'
	  
	  
	  while read LINE
	  do
	      sed -i "/$(echo ${LINE}|grep -oP '(?<=[^a-z])[^ ]*(?= *=)') =/d; $ a${LINE}" /etc/sysctl.conf
	  done <<-EOF
	  # 基础优化
	  kernel.sysrq = 1
	  kernel.hung_task_timeout_secs = 240
	  kernel.panic_on_oops = 1
	  kernel.watchdog_thresh = 50
	  kernel.hardlockup_panic = 1
	  kernel.unprivileged_bpf_disabled = 1
	  net.ipv4.neigh.default.gc_stale_time = 120
	  net.ipv4.conf.all.rp_filter = 0
	  net.ipv4.conf.default.rp_filter = 0
	  net.ipv4.conf.default.arp_announce = 2
	  net.ipv4.conf.lo.arp_announce = 2
	  net.ipv4.conf.all.arp_announce = 2
	  net.ipv4.tcp_max_tw_buckets = 262144
	  net.ipv4.tcp_syncookies = 1
	  net.ipv4.tcp_synack_retries = 2
	  net.ipv4.tcp_slow_start_after_idle = 0
	  net.bridge.bridge-nf-call-iptables = 1
	  net.bridge.bridge-nf-call-ip6tables = 1
	  
	  # hadoop 相关优化
	  net.ipv6.conf.all.disable_ipv6 = 1
	  net.ipv6.conf.default.disable_ipv6 = 1
	  net.ipv6.conf.io.disable_ipv6 = 1
	  
	  vm.swappiness = 0
	  
	  fs.file-mx = 6815744                          # 文件描述符总数
	  fs.aio-max-nr = 1048576                       # 最大并发I/O请求数
	  net.core.rmem_default = 262144                # 操作系统接收缓冲区的默认大小
	  net.core.wmem_default = 262144                # 操作系统发送缓冲区的默认大小
	  net.core.rmem_max = 16777216                  # 系统接收缓冲区最大值
	  net.core.wmem_max = 16777216                  # 系统发送缓冲区最大值
	  EOF
	  
	  ansible all -m shell -a 'sysctl -w net.ipv4.tcp_wmem="4096 262144 16777216"'
	  ansible all -m shell -a 'sysctl -w net.ipv4.tcp_rmem="4096 262144 16777216"'
	  
	  ansible agent -m copy -a 'src=/etc/sysctl.conf dest=/etc/sysctl.conf'
	  ansible all -m shell -a "swapoff -a"
	  ansible all -m shell -a "sed -ri 's/.*swap.*/#&/' /etc/fstab"
	  ansible all -m shell -a "sysctl -p"
	  ```
-
- ssh 免密访问
	- ```bash
	  test -f "$HOME/.ssh/id_rsa" || ssh-keygen -t rsa -C "`hostname`" -f "$HOME/.ssh/id_rsa" -P "" -q
	  
	  test -f "$HOME/.ssh/authorized_keys" || cp /root/.ssh/id_rsa.pub /root/.ssh/authorized_keys
	  chmod 700 /root/.ssh
	  chmod 600 /root/.ssh/authorized_keys
	  
	  ansible agent -m copy -a 'src=/root/.ssh/authorized_keys dest=/root/.ssh/authorized_keys'
	  
	  ```
-
- 时间同步 (无互联网需配置时间服务器)
	- [[Chrony 时间同步]]
	- ```bash
	  ansible all -m yum -a 'name=chrony state=latest'
	  ansible all -m service -a 'name=chronyd state=started enabled=yes'
	  ansible all -m shell -a "timedatectl set-timezone Asia/Shanghai"
	  ansible all -m shell -a "chronyc sources"
	  ```
-
- 网络配置 `vi /etc/sysconfig/network`  可通过 ansible-playbook 模板批量设置
	- ```bash
	  NETWORKING=yes
	  HOSTNAME=<fully.qualified.domain.name>
	  
	  
	  # vi /root/network.j2
	  NETWORKING=yes
	  HOSTNAME={{ hostname|quote }}
	  
	  
	  # vi set_network.yml
	  ---
	  - hosts: all
	    remote_user: root
	    tasks:
	    - name: Set a network
	      template: src=/root/network.j2 dest=/etc/sysconfig/network
	  ```
-
- 防火墙配置
	- ```bash
	  ansible all -m service -a 'name=firewalld state=stopped enabled=no'
	  
	  # 开启的情况
	  firewall-cmd --add-source=ip --permanent --zone=trusted
	  firewall-cmd --reload
	  ```
-
- selinux 和 umask
	- ```bash
	  ansible all -m shell -a "setenforce 0"
	  ansible all -m shell -a 'sed -i "s/^SELINUX=.*$/SELINUX=disabled/" /etc/selinux/config'
	  
	  ansible all -m shell -a "echo 'umask 0022' >> /etc/profile"
	  ```
-
- 重启服务器
-
- MySQL 数据库
	- ```my.cnf
	  # 此配置在 ranger 安装期间设置即可，完成后可关闭。非主从模式可开启或不设置。
	  log_bin_trust_function_creators = 1 # 控制是否可以信任存储函数创建者，不会创建写入二进制日志引起不安全事件的存储函数导致数据不一致
	  ```
	- ```bash
	  # 安装 MySQL
	  yum localinstall \
	  https://dev.mysql.com/get/mysql57-community-release-el7-8.noarch.rpm
	  
	  yum install mysql-community-server
	  
	  systemctl enable mysqld.service --now
	  
	  # 设置密码
	  grep 'A temporary password is generated for root@localhost' \
	  /var/log/mysqld.log |tail -1
	  
	  
	  # 初始化用户和数据库
	  mysql -u root -p
	  
	  -- 密码验证策略低要求(0 或 LOW 代表低级)
	  set global validate_password_policy = 'LOW';
	  -- 密码长度
	  set global validate_password_length = 6;
	  
	  ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';
	  GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost' WITH GRANT OPTION;
	  FLUSH PRIVILEGES;
	  
	  
	  create database hive;
	  create database oozie;
	  create database ambari;
	  
	  CREATE USER 'hive'@'%' IDENTIFIED BY 'R12$%34qw';
	  CREATE USER 'oozie'@'%' IDENTIFIED BY 'R12$%34qw';
	  CREATE USER 'ambari'@'%' IDENTIFIED BY 'bigdata';
	  
	  GRANT ALL PRIVILEGES ON hive.* TO 'hive'@'%' WITH GRANT OPTION;
	  GRANT ALL PRIVILEGES ON oozie.* TO 'oozie'@'%' WITH GRANT OPTION;
	  GRANT ALL PRIVILEGES ON ambari.* TO 'ambari'@'%' WITH GRANT OPTION;
	  
	  
	  -- 配置 SAM 和 schema registry
	  create database registry;
	  create database streamline;
	  
	  CREATE USER 'registry'@'%' IDENTIFIED BY 'R12$%34qw';
	  CREATE USER 'streamline'@'%' IDENTIFIED BY 'R12$%34qw';
	  
	  GRANT ALL PRIVILEGES ON registry.* TO 'registry'@'%' WITH GRANT OPTION ;
	  GRANT ALL PRIVILEGES ON streamline.* TO 'streamline'@'%' WITH GRANT OPTION ;
	  
	  -- 配置 druid 和 superset
	  CREATE DATABASE druid DEFAULT CHARACTER SET utf8;
	  CREATE DATABASE superset DEFAULT CHARACTER SET utf8;
	  
	  CREATE USER 'druid'@'%' IDENTIFIED BY '9oNio)ex1ndL';
	  CREATE USER 'superset'@'%' IDENTIFIED BY '9oNio)ex1ndL';
	  
	  GRANT ALL PRIVILEGES ON druid.* TO 'druid'@'%' WITH GRANT OPTION;
	  GRANT ALL PRIVILEGES ON superset.* TO 'superset'@'%' WITH GRANT OPTION;
	  
	  
	  commit;
	  FLUSH PRIVILEGES;
	  ```
-
- Ranger
	- >Ranger是Hadoop平台的集中式安全管理框架，能够为hadoop平台组件提供细粒度的访问控制。通过Ranger, Hadoop管理员能够轻松地管理各种安全策略，包括：访问文件/文件夹，数据库，Hive表，列, Hbase, YARN等。此外，Ranger还能进行审计管理，以及策略分析，从而为Hadoop环境的深层次分析提供支持。
	- ```sql
	  CREATE USER 'rangerdba'@'localhost' IDENTIFIED BY 'rangerdba';
	  GRANT ALL PRIVILEGES ON *.* TO 'rangerdba'@'localhost';
	  CREATE USER 'rangerdba'@'%' IDENTIFIED BY 'rangerdba';
	  GRANT ALL PRIVILEGES ON *.* TO 'rangerdba'@'%';
	  GRANT ALL PRIVILEGES ON *.* TO 'rangerdba'@'localhost' WITH GRANT OPTION;
	  GRANT ALL PRIVILEGES ON *.* TO 'rangerdba'@'%' WITH GRANT OPTION;
	  FLUSH PRIVILEGES;
	  ```
-
- 使用在线源
	- ```bash
	  wget -O /etc/yum.repos.d/mosga.repo https://makeopensourcegreatagain.com/rpms/mosga.repo
	  ```
-
- 安装 ambari-server 和数据库驱动 （建议使用 8.0 的驱动包）
	- ```bash
	  yum install ambari-server -y
	  # 无互联网则将驱动复制至 /usr/share/java/mysql-connector-java.jar
	  yum install mysql-connector-java* -y
	  
	  # 登入 MySQL
	  mysql> use ambari;
	  mysql> source /var/lib/ambari-server/resources/Ambari-DDL-MySQL-CREATE.sql
	  ```
-
- 安装 Java (有互联网 ambari 可以自动下载)
	- ```bash
	  # 不要使用 8u271 ，会导致 socket leak
	  wget http://public-repo-1.hortonworks.com/ARTIFACTS/jdk-8u112-linux-x64.tar.gz
	  cp jdk-8u112-linux-x64.tar.gz /var/lib/ambari-server/resouces/
	  mkdir /usr/jdk64
	  tar -xaf jdk-8u112-linux-x64.tar.gz -C /usr/jdk64/
	  ```
-
- 配置 ambari
	- ```bash
	  ambari-server setup
	  ```
-
- 删除 smartsense
	- ```bash
	  find /var/lib/ -name "SMARTSENSE" | xargs rm -rf
	  ```
-
- 启动 ambari
	- ```bash
	  ambari-server start
	  
	  # 设置驱动
	  ambari-server setup --jdbc-db=mysql --jdbc-driver=/usr/share/java/mysql-connector-java.jar
	  ```
-
- 访问 web
	- http://<your.ambari.server>:8080
	- 账户密码：admin   /  admin
-
- 页面配置参考安装手册第 6.3 节
	- > https://docs.cloudera.com/HDPDocuments/Ambari-2.7.4.0/bk_ambari-installation/bk_ambari-installation.pdf