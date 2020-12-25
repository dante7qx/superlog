## XXL-JOB





```shell
## 1. 启动数据库
docker start dante-mysql5
## 2. 初始化数据库
下载 https://github.com/xuxueli/xxl-job/blob/master/doc/db/tables_xxl_job.sql
## 3. 启动调度中心，xxl-job-admin
docker run -d --name dante-xxljob -e PARAMS="--spring.datasource.url=jdbc:mysql://docker.for.mac.host.internal:3306/xxl_job?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai --spring.datasource.username=root --spring.datasource.password=iamdante" -p 8600:8080 xuxueli/xxl-job-admin:2.2.0
```





参考文档：

- https://www.xuxueli.com/xxl-job/