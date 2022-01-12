- ## CentOS7 minimal 配置桌面并安装 VNC
- 安装 GNOME Desktop
  ```bash
  yum groupinstall -y "X Window System"
  yum install -y gnome-classic-session gnome-terminal nautilus-open-terminal control-center liberation-mono-fonts
  unlink /etc/systemd/system/default.target
  ln -sf /lib/systemd/system/graphical.target /etc/systemd/system/default.target
  reboot
  ```
- 安装 VNC
  ```bash
  yum install -y tigervnc-server
  
  # 修改文件中的 USER 为实际用户
  vi /lib/systemd/system/vncserver@.service 
  cp /lib/systemd/system/vncserver@.service /etc/systemd/system/
  
  systemctl daemon-reload
  systemctl enable vncserver@:1.service --now
  
  # 设置 vnc 密码
  vncpasswd ~/.vnc/passwd
  
  # would you like enter a viewer only password?
  # 设置一个只能看不能操作的密码
  ```
- 服务参考文件: `/etc/systemd/system/vncserver@.service`
  ```bash
  [Unit]
  Description=Remote desktop service (VNC)
  After=syslog.target network.target
  
  [Service]
  Type=simple
  
  # Clean any existing files in /tmp/.X11-unix environment
  ExecStartPre=/bin/sh -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'
  # 修改此处为实际用户
  ExecStart=/usr/bin/vncserver_wrapper root %i
  ExecStop=/bin/sh -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'
  
  [Install]
  WantedBy=multi-user.target
  ```
-
- ## 无桌面图形化
- ```bash
  yum install -y xterm libXp libXtst-devel
  yum install -y Xvfb
  yum install -y x11vnc
  
  # apt-get install -y x11vnc
  
  # 启动  端口：12345   密码：password
  x11vnc -rfbport 12345 -passwd password -create -forever
  
  
  # 可显示图片
  yum install -y xloadimage
  
  xview picture.png
  ```