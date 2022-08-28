- https://www.postgresql.org/docs/current/index.html
- PostgreSQL 是对象关系数据库管理系统，MySQL 是关系型数据库管理系统
- 首先要理解 **对象关系** 的概念，才能明白 PostgreSQL 库和模式，用户和角色的定义。
- # 安装
- ## 基于 rpm
- ```bash
  sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
  sudo yum install -y postgresql14-server
  sudo /usr/pgsql-14/bin/postgresql-14-setup initdb
  sudo systemctl enable postgresql-14
  sudo systemctl start postgresql-14
  
  # 卸载
  systemctl stop postgresql-14
  yum remove postgresql postgresql14-libs
  rm -rf /var/lib/pgsql/
  rm -rf /usr/pgsql-14/
  
  yum list installed | grep postgres
  ```
- ## From source
- ```bash
  curl -o postgresql-14.4.tar.gz  https://ftp.postgresql.org/pub/source/v14.4/postgresql-14.4.tar.gz
  gunzip postgresql-14.4.tar.gz
  tar xf postgresql-14.4.tar
  cd postgresql-14.4
  yum -y install gcc-c++ readline-devel zlib-devel
  ./configure
  make
  su
  make install
  adduser postgres
  mkdir /usr/local/pgsql/data
  chown postgres /usr/local/pgsql/data
  su - postgres
  /usr/local/pgsql/bin/initdb -D /usr/local/pgsql/data
  /usr/local/pgsql/bin/pg_ctl -D /usr/local/pgsql/data -l logfile start
  /usr/local/pgsql/bin/createdb test
  /usr/local/pgsql/bin/psql test
  ```
- # 使用
- ## 常用命令
- ```sql
  \q 退出
  \h 帮助 \h select
  \? 查看命令列表
  \l 列出数据库
  \c 连接其他数据库
  \d 列出表 \d table_name 展示表结构
  \du 列出用户
  \password 修改密码
  \! 执行 shell 命令
  ```
- ## 创建数据库
- ```bash
  createdb mydb
  /usr/local/pgsql/bin/dropdb mydb
  
  psql mydb
  
  
  SELECT version();
  SELECT current_date;
  SELECT 2 + 2;
  \q
  ```
- ## DDL
- ```sql
  CREATE TABLE weather (
      city            varchar(80),
      temp_lo         int,           -- low temperature
      temp_hi         int,           -- high temperature
      prcp            real,          -- precipitation
      date            date
  );
  
  CREATE TABLE cities (
      name            varchar(80),
      location        point
  );
  
  DROP TABLE tablename;
  ```
- ## DML
- ```sql
  INSERT INTO weather VALUES ('San Francisco', 46, 50, 0.25, '1994-11-27');
  INSERT INTO cities VALUES ('San Francisco', '(-194.0, 53.0)');
  INSERT INTO weather (city, temp_lo, temp_hi, prcp, date)
      VALUES ('San Francisco', 43, 57, 0.0, '1994-11-29');
  INSERT INTO weather (date, city, temp_hi, temp_lo)
      VALUES ('1994-11-29', 'Hayward', 54, 37);
  
  -- 从服务器加载数据，比插入更快。
  -- COPY weather FROM '/home/user/weather.txt';
  
  
  SELECT * FROM weather;
  SELECT city, temp_lo, temp_hi, prcp, date FROM weather;
  SELECT city, (temp_hi+temp_lo)/2 AS temp_avg, date FROM weather;
  SELECT * FROM weather
      WHERE city = 'San Francisco' AND prcp > 0.0;
  SELECT * FROM weather
      ORDER BY city;
  SELECT * FROM weather
      ORDER BY city, temp_lo;
  SELECT DISTINCT city
      FROM weather;
  SELECT DISTINCT city
      FROM weather
      ORDER BY city;
  
  
  SELECT * FROM weather JOIN cities ON city = name;
  SELECT city, temp_lo, temp_hi, prcp, date, location
      FROM weather JOIN cities ON city = name;
  SELECT weather.city, weather.temp_lo, weather.temp_hi,
         weather.prcp, weather.date, cities.location
      FROM weather JOIN cities ON weather.city = cities.name;
  SELECT *
      FROM weather, cities
      WHERE city = name;
  SELECT *
      FROM weather LEFT OUTER JOIN cities ON weather.city = cities.name;
  SELECT w1.city, w1.temp_lo AS low, w1.temp_hi AS high,
         w2.city, w2.temp_lo AS low, w2.temp_hi AS high
      FROM weather w1 JOIN weather w2
          ON w1.temp_lo < w2.temp_lo AND w1.temp_hi > w2.temp_hi;
  SELECT *
      FROM weather w JOIN cities c ON w.city = c.name;
  
  
  SELECT max(temp_lo) FROM weather;
  SELECT city FROM weather
      WHERE temp_lo = (SELECT max(temp_lo) FROM weather);
  SELECT city, max(temp_lo)
      FROM weather
      GROUP BY city
      HAVING max(temp_lo) < 40;
  SELECT city, max(temp_lo)
      FROM weather
      WHERE city LIKE 'S%'            -- (1)
      GROUP BY city
      HAVING max(temp_lo) < 40;
  
  
  UPDATE weather
      SET temp_hi = temp_hi - 2,  temp_lo = temp_lo - 2
      WHERE date > '1994-11-28';
  
  
  DELETE FROM weather WHERE city = 'Hayward';
  
  -- DELETE FROM tablename;
  ```
- ## DCL
- ```sql
  -- 创建的角色不能登录
  CREATE ROLE name;
  -- 创建的用户可以登录
  CREATE USER name;
  DROP ROLE name;
  
  -- 命令行操作
  -- createuser name
  -- dropuser name
  
  SELECT rolname FROM pg_roles;
  
  create user dawn with password '123456';
  
  -- 查看用户
  \du
  
  -- 修改密码
  alter user 'postgres' with password 'password';
  
  -- 授权
  grant all privileges on database 'dbname' to 'username';
  
  -- 切换到要访问的数据库
  \c mydb
  grant all privileges on all tables in schema public to 'username';
  
  -- 取消授权
  revoke all privileges on all tables in schema public from 'username';
  revoke all privileges on database 'mydb' from 'username';
  drop user 'username';
  ```
-
- ## 使用客户端访问
- ```bash
  # 修改可访问 IP
  vi data/pg_hba.conf +91
  host    all             all             0.0.0.0/0            trust
  # 修改监听地址
  vi data/postgresql.conf +60
  listen_addresses = '*'          # what IP address(es) to listen on;
  
  # 重启
  pg_ctl -D /usr/local/pgsql/data restart
  ```