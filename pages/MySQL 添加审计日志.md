- 插件对数据库性能有影响。
- ---
- ## 插件 【 server_audit.so 】
- [mariadb-10.5.13](https://mirrors.aliyun.com/mariadb//mariadb-10.5.13/bintar-linux-systemd-x86_64/mariadb-10.5.13-linux-systemd-x86_64.tar.gz) 在`lib/plugin/server_audit.so` 将插件文件取出
- 查看当前数据库插件目录位置 `SHOW GLOBAL VARIABLES LIKE '%plugin_dir%';`
- 将 server_audit.so 放至 `/usr/lib64/mysql/plugin`
- `chmod +x /usr/lib64/mysql/plugin/server_audit.so`
- `INSTALL PLUGIN server_audit SONAME 'server_audit.so';`
- `ERROR 1126 (HY000): Can't open shared library '/usr/lib64/mysql/plugin/server_audit.so' (errno: 13 /usr/lib64/mysql/plugin/server_audit.so: undefined symbol: psi_prlock_wrlock)`
  可能是版本不兼容
- 新版本数据库加载审计插件后无法启动
  background-color:: #793e3e
- `show variables like '%audit%';`
- `set global server_audit_logging=on;`
- 在 my.cnf 中新增 `server_audit_logging=on`
- ---
- ## 插件 【mysql-audit】
- [mysql-audit](https://github.com/mcafee/mysql-audit/wiki)
- 查看插件目录
  ```bash
  mysql> show global variables like 'plugin_dir';
  +---------------+--------------------------+
  | Variable_name | Value                    |
  +---------------+--------------------------+
  | plugin_dir    | /usr/lib64/mysql/plugin/ |
  +---------------+--------------------------+
  1 row in set (0.01 sec)
  ```
- 将文件上传至 `/usr/lib64/mysql/plugin/` 并授权 `chmod +x /usr/lib64/mysql/plugin/libaudit_plugin.so`
- 将 `offset-extract.sh` 上传至容器中，执行 `yum install which gdb && sh offset-extract.sh /usr/sbin/mysqld`
- 在 `my.cnf` 中添加 `plugin-load=AUDIT=libaudit_plugin.so `
  或者 `INSTALL PLUGIN AUDIT SONAME 'libaudit_plugin.so';`
- 验证
  `show plugins;`
  `show global status like 'AUDIT_version';`
  `set global audit_json_file=ON;`
  执行任一 SQL 查看日志 /var/lib/mysql/mysql-audit.json
- `my.cnf` 配置
  ```cnf
  plugin-load=AUDIT=libaudit_plugin.so 
  audit_json_file=on
  audit_json_log_file=/var/lib/mysql/mysql-audit.json
  # 替换为 offset-extract.sh 执行的结果
  audit_offsets=7824, 7872, 3632, 4792, 456, 360, 0, 32, 64, 160, 536, 7988, 4360, 3648, 3656, 3660, 6072, 2072, 8, 7056, 7096, 7080, 13472, 148, 672
  ```
-