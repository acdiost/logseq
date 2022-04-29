- ```bash
  # 权限问题可指定用户
  export HADOOP_USER_NAME=hdfs
  
  # 执行查询，查询语句不要带分号
  sqoop eavl --connect jdbc:mysql://127.0.0.1/test \
  --username root \
  --password 123456 \
  --query "select * from test.test_table"
  
  sqoop eavl --connect jdbc:mysql://127.0.0.1/test \
  --username root \
  --password 123456 \
  --query "show create table test.test_table"
  
  sqoop eavl --connect jdbc:mysql://127.0.0.1/test \
  --username root \
  --password 123456 \
  --query "desc test.test_table"
  
  
  sqoop eavl --connect jdbc:oracle:thin:@//127.0.0.1:1521/orcl \
  --username scott \
  --password tiger \
  --query "select * from EMP"
  
  sqoop eavl --connect jdbc:oracle:thin:@//127.0.0.1:1521/orcl \
  --username scott \
  --password tiger \
  --query "select * from user_tab_columns where table_name='EMP'"
  ```
-
- 抽取脚本：
	- ```bash
	  #!/usr/bin/env bash
	  # author: dawn
	  # date: 2022-04-24
	  # desc: sqoop 增量抽取方案: 全量抽取 字段增量（字段类型必须是整型） 时间增量
	  # crontab: 10 * * * * bash sqoop_(taskname).sh
	  # version: 1.0.0
	  
	  
	  # 环境变量
	  HADOOP_USER_NAME=hdfs                           # 抽取用户，以解决 sqoop 抽取时报 hdfs 权限不足
	  
	  # 数据源配置
	  DB_CONNECTION='jdbc:oracle:thin:@//127.0.0.1:1521/orcl'
	  # 需要修改抽取查询条件
	  #DB_CONNECTION='jdbc:mysql://localhost:3306/test'
	  DB_USERNAME='user'
	  DB_PASSWORD='pwd'
	  DB_DATABASE='db'                                # 要抽取的库名
	  DB_TABLE='table'                                # 要抽取的表名
	  
	  # hive 配置
	  HIVE_DATABASE='hive_db'                         # hive 库名
	  HIVE_TABLE='hive_'${DB_TABLE,,}                 # hive 表名转换成小写
	  
	  # hdfs 配置
	  HIVE_PATH='/user/hive/warehouse'                # hive 在 hdfs 的路径
	  
	  # impala 配置
	  IMPALA_USER='hive'                              # impala 认证用户
	  
	  # sqoop 配置
	  DELIMITER='\0030'                               # 分隔符 在 hive 中将转换为：u0018
	  NUM_MAPPERS='1'                                 # map reduce 线程数默认 4,当该值大于 1 则必须指定 --split-by 参数（参数默认为表主键，若无则可：select * from (select ROWNUM AS pk, T.* from xxx T) where \$CONDITIONS "）
	  TARGET_DIR='/tmp/hive/'${DB_TABLE}              # 数据临时存放到 hdfs 的目录路径
	  
	  SQOOP_FILE=$(su - hdfs -c "mktemp")             # 临时存放 sqoop 抽取配置
	  SQOOP_LOGS='/var/log/sqoop/sqoop.log'           # sqoop 抽取日志记录
	  SQOOP_PID="/var/run/sqoop_${DB_TABLE,,}.pid"    # 当前 sqoop 抽取进程
	  SQOOP_TIMEOUT=1800                              # sqoop 抽取任务超时时间 30 分钟，单位秒
	  
	  # 抽取配置
	  INCREMENTAL_FIELD='columns'                     # 增量字段为日期格式 date
	  SELECT_FIELD='columns, columns, columns'        # 要抽取的字段，以英文逗号分隔
	  TIMESTAPMS_TIME='1970-01-01 00:00:00'           # 纪元时间，未使用
	  START_TIME='2022-04-24 00:00:00'                # 指定抽取起始时间
	  END_TIME=$(date +"%Y-%m-%d %H:%M:%S")           # 指定抽取结束时间
	  
	  
	  # 检测当前脚本是否还在运行
	  function sqoop_pid(){
	      if [ -f ${SQOOP_PID} ];then
	          # 如果 pid 存在，检查超时
	          last_change_time=$(stat -c %Z ${SQOOP_PID})
	          current_time=$(date +%s)
	          time_sub=$((${current_time} - ${last_change_time}))
	          if [ $time_sub -gt ${SQOOP_TIMEOUT} ];then
	              # 超时，杀死进程重新执行
	              cat ${SQOOP_PID} | xargs kill -9
	              # 生成本次运行的 pid
	              echo $$ > ${SQOOP_PID}
	              return 0
	          else
	              # 未超时，直接退出
	              echo '抽取任务仍在执行，退出本次运行任务，建议调整该任务调度间隔时间' >> ${SQOOP_LOGS}
	              exit 1
	          fi
	      else
	          # pid 不存在,直接执行抽取任务
	          echo $$ > ${SQOOP_PID}
	          return 0
	      fi
	  }
	  
	  # 增量抽取
	  function incremental_extract(){
	      echo '抽取模式：增量抽取' >> ${SQOOP_LOGS}
	      echo "抽取任务： ${DB_DATABASE}.${DB_TABLE}" >> ${SQOOP_LOGS}
	      # 通过 hive 查询时间
	      hive_result=$(su - hdfs -c "hive -S -e \"select max(${INCREMENTAL_FIELD}) from ${HIVE_DATABASE}.${HIVE_TABLE} \"")
	      HIVE_TIME=$(echo ${hive_result:0:-2})
	      # 查询资源库时间
	      ziyuanku_result=$(sqoop eval --connect ${DB_CONNECTION} --username ${DB_USERNAME} --password ${DB_PASSWORD} --query " select max(${INCREMENTAL_FIELD}) from ${DB_DATABASE}.${DB_TABLE} ")
	      ZIYUANKU_TIME=$(echo ${ziyuanku_result} | awk -F\| '{print $4}' | xargs echo)
	  
	      python_script="
	  print("\'${ZIYUANKU_TIME}\'" > "\'${HIVE_TIME}\'")
	  "
	      result=$(python -c """${python_script}""")
	  
	      if [ ${result} = 'True' ];then
	          # 资源库时间大于 hive 库
	          # 抽取文件配置
	          # --hive-drop-import-delims 删除 hive 默认的分隔符
	          # --null-string 和 --null-non-string 将空数据替换为空白字符
	          cat <<EOF | tee ${SQOOP_FILE}
	  import
	  --connect
	  ${DB_CONNECTION}
	  --username
	  ${DB_USERNAME}
	  --password
	  ${DB_PASSWORD}
	  --query
	  " select ${SELECT_FIELD:-*} from ${DB_DATABASE}.${DB_TABLE} where ${INCREMENTAL_FIELD} > to_date('${HIVE_TIME}', 'YYYY-MM-DD HH24:MI:SS') and \$CONDITIONS "
	  --hive-import
	  --target-dir
	  ${TARGET_DIR}
	  --delete-target-dir
	  --hive-database
	  ${HIVE_DATABASE}
	  --hive-table
	  ${HIVE_TABLE}
	  --fields-terminated-by
	  '${DELIMITER}'
	  --null-string
	  ''
	  --null-non-string
	  ''
	  --hive-drop-import-delims
	  -m
	  ${NUM_MAPPERS:-1}
	  
	  EOF
	  
	          # 执行抽取文件并输入至日志文件
	          echo '上次抽取记录时间最大值: '${HIVE_TIME} >> ${SQOOP_LOGS}
	          echo '本次资源库记录时间最大值: '${ZIYUANKU_TIME} >> ${SQOOP_LOGS}
	          temp_log=$(su - hdfs -c "mktemp")
	          su - hdfs -c "sqoop --options-file ${SQOOP_FILE}" &>> ${temp_log}
	          # 获取抽取记录条数
	          records_input=$(cat ${temp_log} | grep 'input records=' | awk -F= '{print $2}')
	          records_output=$(cat ${temp_log} | grep 'output records=' | awk -F= '{print $2}')
	          rm -f ${temp_log}
	          echo "本次共抽取 input records: ${records_input:-0} , output records: ${records_output:-0}" >> ${SQOOP_LOGS}
	          # 删除抽取文件配置
	          rm -f ${SQOOP_FILE}
	      else
	          # 资源库时间小于等于 hive 库，跳过抽取任务
	          echo 'hive 库记录时间值: '${HIVE_TIME} >> ${SQOOP_LOGS}
	          echo '资源库记录时间值: '${ZIYUANKU_TIME} >> ${SQOOP_LOGS}
	          echo 'no datas update, 资源库未更新数据，跳过此次抽取任务。' >> ${SQOOP_LOGS}
	      fi
	  
	  }
	  
	  
	  # 全量抽取
	  function full_extract(){
	      echo '抽取模式：全量抽取' >> ${SQOOP_LOGS}
	      echo "抽取任务： ${DB_DATABASE}.${DB_TABLE}" >> ${SQOOP_LOGS}
	      # 1. 清理文件
	      echo '开始删除 hdfs 数据文件' >> ${SQOOP_LOGS}
	      su - hdfs -c "hdfs dfs -rm -r -skipTrash ${HIVE_PATH}/${HIVE_DATABASE}.db/${HIVE_TABLE}"
	      echo '开始删除 hive 表' >> ${SQOOP_LOGS}
	      su - hdfs -c "hive -S -e \"drop table ${HIVE_DATABASE}.${HIVE_TABLE}\" "
	      # 2. 抽取数据
	      echo '开始抽取数据' >> ${SQOOP_LOGS}
	      cat <<EOF | tee ${SQOOP_FILE}
	  import
	  --connect
	  ${DB_CONNECTION}
	  --username
	  ${DB_USERNAME}
	  --password
	  ${DB_PASSWORD}
	  --query
	  " select ${SELECT_FIELD:-*} from ${DB_DATABASE}.${DB_TABLE} where \$CONDITIONS "
	  --hive-import
	  --hive-overwrite 
	  --target-dir
	  ${TARGET_DIR}
	  --delete-target-dir
	  --hive-database
	  ${HIVE_DATABASE}
	  --hive-table
	  ${HIVE_TABLE}
	  --fields-terminated-by
	  '${DELIMITER}'
	  --null-string
	  ''
	  --null-non-string
	  ''
	  --hive-drop-import-delims
	  -m
	  ${NUM_MAPPERS:-1}
	  
	  EOF
	      # 执行抽取文件并输入至日志文件
	      temp_log=$(su - hdfs -c "mktemp")
	      su - hdfs -c "sqoop --options-file ${SQOOP_FILE}" &>> ${temp_log}
	      # 获取抽取记录条数
	      records_input=$(cat ${temp_log} | grep 'input records=' | awk -F= '{print $2}')
	      records_output=$(cat ${temp_log} | grep 'output records=' | awk -F= '{print $2}')
	      rm -f ${temp_log}
	      echo "本次共抽取 input records: ${records_input:-0} , output records: ${records_output:-0}" >> ${SQOOP_LOGS}
	      # 删除抽取文件配置
	      rm -f ${SQOOP_FILE}
	      # 3. 刷新 impala 元数据
	      impala-shell -u ${IMPALA_USER} -q "invalidate metadata ${HIVE_DATABASE}.${HIVE_TABLE}"
	  }
	  
	  
	  ############### TODO ###############
	  
	  # 数据补录
	  function data_supplement(){
	      supplement_condition="id='xxxx'"
	  cat <<EOF | tee ${SQOOP_FILE}
	  import
	  --connect
	  ${DB_CONNECTION}
	  --username
	  ${DB_USERNAME}
	  --password
	  ${DB_PASSWORD}
	  --query
	  " select ${SELECT_FIELD:-*} from ${DB_DATABASE}.${DB_TABLE} where ${supplement_condition} and \$CONDITIONS "
	  --hive-import
	  --target-dir
	  ${TARGET_DIR}
	  --delete-target-dir
	  --hive-database
	  ${HIVE_DATABASE}
	  --hive-table
	  ${HIVE_TABLE}
	  --fields-terminated-by
	  '${DELIMITER}'
	  --null-string
	  ''
	  --null-non-string
	  ''
	  --hive-drop-import-delims
	  -m
	  ${NUM_MAPPERS:-1}
	  
	  EOF
	  }
	  
	  
	  # hive 查询结果导出到 csv
	  function export_data(){
	      hive -S -e "set hive.cli.print.header=true; select * from ${HIVE_DATABASE}.${HIVE_TABLE}"  | sed 's/[\t]/,/g' >> ./hive.csv
	  }
	  
	  
	  # 创建分区
	  function create_partition(){
	      set hive.exec.dynamic.partition=true                # 开启动态分区
	      set hive.exec.dynamic.partition.mode=nonstrict      # 非严格模式
	  
	  }
	  
	  
	  # 合并
	  function sqoop_merge(){
	      sqoop codegen --connect ${DB_CONNECTION} --username ${DB_USERNAME} --password ${DB_PASSWORD} --table ${DB_TABLE}
	      sqoop merge --new-data ${HIVE_PATH}/${HIVE_DATABASE}.db/${HIVE_TABLE} --onto ${HIVE_PATH}/${HIVE_DATABASE}.db/${HIVE_TABLE} --target-dir ${TARGET_DIR} --jar-file xxx.jar --class-name xxx --merge-key xxx
	  }
	  
	  
	  main (){
	      echo 'sqoop tasks startup' >> ${SQOOP_LOGS}
	      date >> ${SQOOP_LOGS}
	      # 判断 pid
	      sqoop_pid
	      if [ $? == 0 ];then
	          # 增量抽取
	          incremental_extract
	          # 全量抽取
	          # full_extract
	      fi
	      rm -f ${SQOOP_PID}
	      date >> ${SQOOP_LOGS}
	      echo -e 'sqoop tasks done\n' >> ${SQOOP_LOGS}
	  }
	  
	  
	  main $@
	  
	  ```