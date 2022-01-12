- ## 使用公钥登录
- 推荐 4096 位
  `ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -C "xxx key"`
- 拷贝公钥至服务器
  `ssh-copy-id -i /path/public-key user@host`
- ## 禁止用户名密码登录
  ```bash
  # 禁止 root 通过 ssh 登录
  PermitRootLogin no
  
  # 禁止使用用户名密码登录
  ChallengeResponseAuthentication no
  PasswordAuthentication no
  UsePAM no
  
  AuthenticationMethods publickey
  PubkeyAuthentication yes
  
  # 禁用空密码
  PermitEmptyPasswords no
  ```
- ## 允许部分用户使用 ssh
  ```bash
  AllowUsers user0 user1 user2
  AllowGroups group0
  ```
- ## 修改默认端口
  ```bash
  Port 54321
  ```
- ## 明确 ssh 可连接的地址
  ```bash
  ListenAddress 192.168.1.1
  ListenAddress 172.16.1.1
  ```
- ## 设置 ssh 超时
  ```bash
  # 300 秒
  ClientAliveInterval 300
  # 如果客户端没有响应则判断一次超时，设置允许超时的次数
  ClientAliveCountMax 0
  ```
- ## 防火墙限制 IP
  只允许来自 192.168.1 网段的连接请求
  ```bash
  ufw allow from 192.168.1.0/24 to any port 54321
  ```