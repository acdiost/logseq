- ## Oracle
- ```bash
  sqlplus -S 连接信息 > filepath << EOF
  set heading off
  set line 400
  col c1 for a30
  col c2 for a30
  
  select inst_id, username, cnt, sum(cnt) over() cnts from
  (select inst_id, username, count(1) cnt from gv$session where inst_id=3 group by inst_id,username)
  group by inst_id, username, cnt
  order by 1, 3 des;
  
  exit
  EOF
  ```