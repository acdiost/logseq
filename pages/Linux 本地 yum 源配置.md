- 互联网获取依赖包
- ### 添加第三方源
	- ```bash
	  rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
	  wget -O /etc/yum.repos.d/docker-ce.repo https://download.docker.com/linux/centos/docker-ce.repo
	  rpm -ih https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm
	  rpm -Uh https://harbottle.gitlab.io/harbottle-main/7/x86_64/harbottle-main-release.rpm
	  rpm -Uh https://rpms.remirepo.net/enterprise/7/remi/x86_64/remi-release-7.9-1.el7.remi.noarch.rpm
	  ```
- ### 下载所需依赖
	- ```bash
	  yum makecache fast
	  yum install --downloadonly --downloaddir=./rpms createrepo unzip wget nginx ansible chrony nfs-utils lrzsz mysql-community-server docker-ce java
	  ```
- ### 创建元数据文件
	- ```bash
	  yum install -y createrepo
	  createrepo rpms/
	  # 更新元数据文件
	  # createrepo --update rpms/
	  ```
- ### 配置本地源
	- #### 本地文件形式
	  将依赖文件放至 /media/share/ 目录下；其他目录按需更改
	- ```bash
	  cat <<EOF | sudo tee >> /etc/yum.repos.d/local.repo
	  [local]
	  name=Local - Media
	  baseurl=file:///media/share/rpms/
	  gpgcheck=0
	  enabled=1
	  EOF
	  
	  # 测试安装
	  yum install nginx -y
	  ```
	- #### 通过 nginx 配置
		- 目录： /media/share/rpms/ 
		  将依赖文件放在 rpms/ 目录下
		  需要共享的文件可放在 share/ 目录下
		- ```bash
		  cat <<EOF | sudo tee >> /etc/nginx/conf.d/repo.conf
		  server {
		      listen 82;
		      server_name localhost;
		  
		      autoindex on;
		  
		      location / {
		          root /media/share/rpms;
		      }
		      # 可用于文件分享下载，访问方式： http://ip:82/share
		      location /share {
		          root /media;
		      }
		  }
		  EOF
		  
		  systemctl start nginx && systemctl enable nginx
		  
		  ip=''
		  cat <<EOF | sudo tee >> /etc/yum.repos.d/Customize.repo
		  [Customize]
		  name=Customize - Media
		  baseurl=http://$ip:82
		  gpgcheck=0
		  enabled=1
		  EOF
		  
		  # 测试安装
		  yum install docker-ce
		  ```