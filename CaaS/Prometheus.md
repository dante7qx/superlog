## Prometheus









### 三. 安装

#### 1. Prometheus Server

使用 docker 安装

```bash
docker run --name=dante-prom -d \
-p 9190:9090 \
-v /Users/dante/Documents/Technique/Docker/volume/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml \
-v /Users/dante/Documents/Technique/Docker/volume/prometheus/rules.yml:/etc/prometheus/rules.yml \
prom/prometheus:v2.13.0 \
--config.file=/etc/prometheus/prometheus.yml \
--web.enable-lifecycle
```



七. 参考资料

- https://www.cnblogs.com/chenqionghe/p/10494868.html
- https://blog.csdn.net/yangshangwei/article/details/88385783#SpringBootPrometheus_136