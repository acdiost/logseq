- [vagrant 下载地址](https://www.vagrantup.com/downloads)
- 常用命令
- ```bash
  # 安装程序后
  vagrant init generic/centos7
  # vagrant init generic/ubuntu2004
  # 管理员终端
  vagrant up --provider=hyperv --color
  
  # 添加国内镜像
  vagrant box add <name> <url> --provider=
  
  # 启动
  vagrant up
  
  # 查看状态
  vagrant status
  
  # 连接 默认密码：vagrant
  vagrant ssh <name>
  # 查看配置
  vagrant ssh-config
  # 机器的暂停 恢复 重启 关机
  vagrant suspend / resume / reload / halt <name>
  # 删除
  vagrant destroy <name>
  
  # 安装插件
  vagrant plugin list
  vagrant plugin install vagrant-vbguest # 指定版本 --plugin-version 0.21
  
  # 查看下载的 box
  vagrant box list
  
  # 下载 box
  vagrant box add generic/centos7 # 指定参数 --box-version=3.2.24 --provider=hyperv
  
  # 删除
  vagrant remove generic/centos7
  ```
- 创建新主机
- ```ruby
  hosts = [
    {
      :name => "node1",
      :box => "generic/centos7",
      :ip => "192.168.31.201"
    },
    {
      :name => "node2",
      :box => "generic/centos7",
      :ip => "192.168.31.202"
    },
    {
      :name => "node3",
      :box => "generic/centos7",
      :ip => "192.168.31.203"
    }
  ]
  
  Vagrant.configure("2") do |config|
    config.ssh.insert_key = false
    config.vm.box_check_update = false
    
    # 目录为上级目录
    config.vm.provider "virtualbox" do |_, vb|
      vb.vm.synced_folder "../", "/vagrant", type: "virtualbox"
    end
  
    hosts.each do |item|
      config.vm.define item[:name] do |host|
        host.vm.box = item[:box]
        host.vm.hostname = item[:name]
        host.vm.network :public_network, ip: item[:ip]
        host.vm.provider "virtualbox" do |v|
          v.memory = 4096
          v.cpus = 2
          # v.gui = true
        end
      end
    end
  
    config.vm.provision "shell", inline: <<-SHELL
    echo root | passwd --stdin root
    sed -i 's/^#PermitRootLogin yes$/PermitRootLogin yes/' /etc/ssh/sshd_config
    sed -i 's/^PasswordAuthentication no$/PasswordAuthentication yes/' /etc/ssh/sshd_config
    systemctl restart sshd
    ip a
  SHELL
  end
  ```
-