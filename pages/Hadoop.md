-
- ```bash
  hadoop dfs -ls
  hadoop fs -ls
  hdfs dfs -ls
  
  
  hadoop dfs -du -s -h /path
  hadoop fs -du -s -h /path
  hdfs dfs -du -s -h /path
  
  hdfs dfs -put /localpath /hdfspath
  hdfs dfs -moveFromLocal old new
  hdfs dfs -get /hdfspath /localpath
  hdfs dfs -mkdir /test
  hdfs dfs -mv /hdfspath /hdfspath
  hdfs dfs -cp /hdfspath /hdfspath
  hdfs dfs -rm file
  hdfs dfs -rm -r /dirctory/
  hdfs dfs -cat /file.txt
  hdfs dfs -tail -f /file.txt
  hdfs dfs -count /file/
  hdfs dfs -df /
  hdfs dfs -df -h /
  ```