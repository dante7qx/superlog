## MySQL 积累

### 1. mysql + json

https://segmentfault.com/a/1190000011580030

### 2. mysql5.7分组排序

```sql
-- 测试
drop table if exists t_flow;
create table t_flow (
  id bigint(20) not null auto_increment comment 'id',
  def_id bigint(20),
  inst_id bigint(20),
  name varchar(20) default '' NOT NULL comment '名称',
  start_time datetime comment '开始时间',
  end_time datetime comment '结束时间',
  
  primary key (id)
) engine = innodb auto_increment = 1 comment = '流程';

insert into t_flow (def_id, inst_id, name, start_time, end_time) values
(10, 20, '流程1', '2022-03-06 14:36:14.721', '2022-03-06 14:36:40.098'),
(10, 20, '流程2', '2022-03-06 14:40:14.000', '2022-03-06 14:45:40.000'),
(15, 25, '流程3', '2022-03-06 14:50:14.000', '2022-03-06 14:53:40.098'),
(18, 30, '流程4', '2022-03-06 14:58:14.000', '2022-03-06 14:59:40.000')
```

使用子查询，先排序再分组，其中 sql_mode 中要去掉 ONLY_FULL_GROUP_BY

```sql
-- 关掉ONLY_FULL_GROUP_BY
set global sql_mode='STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION';

-- 先排序再分组
select * from (
	select * from t_flow t  where id > 0 order by t.end_time desc
) as k group by k.def_id, k.inst_id;

-- 这个方式在低版本中有效。在 5.7 版本中引入新特性 derived_merge 优化过后无效了。

-- 处理方式：使用 having 来阻止合并
select * from (
	select * from t_flow t  where id > 0 having 1=1 order by t.end_time desc
) as k group by k.def_id, k.inst_id;
```

### 3. 递归查询

https://blog.csdn.net/xubenxismile/article/details/107662209

### 4. 创建用户、授权

```sql
use mysql;

create database testDB default charset utf8mb4 collate utf8mb4_general_ci;

CREATE USER 'hzyingjiju'@'localhost' IDENTIFIED BY 'Risun8768!';
CREATE USER 'hzyingjiju'@'%' IDENTIFIED BY 'Risun8768!';

grant all privileges on testDB.* to 'hzyingjiju'@'localhost' IDENTIFIED BY 'Risun8768!';

grant all privileges on testDB.* to 'hzyingjiju'@'%' IDENTIFIED BY 'Risun8768!';

```

