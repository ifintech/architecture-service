## docker日志处理

## beats

```shell
docker pull docker.elastic.co/beats/metricbeat:5.5.0
```

```
chmod 666 /var/run/docker.sock
```

**metricbeat.yml**

```yaml
metricbeat.modules:
- module: system
  metricsets:
    - cpu
    - filesystem
    - memory
    - network
    - process
  enabled: true
  period: 10s
  processes: ['.*']
  cpu_ticks: false
- module: docker
  metricsets: ["container", "cpu", "diskio", "healthcheck", "info", "memory", "network"]
  hosts: ["unix:///var/run/docker.sock"]
  enabled: true
  period: 10s
output.elasticsearch:
  hosts: ["10.1.2.4:9200"]
  template.name: "metricbeat"
  template.path: "metricbeat.template.json"
  template.overwrite: false
```

```shell
docker config create metricbeat-conf /data1/metricbeat/metricbeat.yml
```

```shell
docker run \
  --name metricbeat \
  -v /proc:/hostfs/proc:ro \
  -v /sys/fs/cgroup:/hostfs/sys/fs/cgroup:ro \
  -v /:/hostfs:ro \
  -v /data1/metricbeat/metricbeat.yml:/usr/share/metricbeat/metricbeat.yml \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --net=host \
  dockerhub.jrmf360.com/elastic/metricbeat:5.5.0 
```

```shell
docker service create --name metricbeat \
  --mode global \
  --config source=metricbeat-conf,target=/usr/share/metricbeat/metricbeat.yml \
  --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  --mount type=bind,src=/proc,dst=/hostfs/proc,ro=true \
  --mount type=bind,src=/sys/fs/cgroup,dst=/hostfs/sys/fs/cgroup,ro=true \
  --mount type=bind,src=/,dst=/hostfs,ro=true \
  --network host \
  dockerhub.jrmf360.com/elastic/metricbeat:5.5.0
```

### 安装kibana 索引及可视化模板

```shell
docker run \
dockerhub.jrmf360.com/elastic/metricbeat:5.5.0 ./scripts/import_dashboards -es http://10.1.2.4:9200 -url https://artifacts.elastic.co/downloads/beats/beats-dashboards/beats-dashboards-5.5.0.zip
```

## Nginx

```shell
setsebool -P httpd_can_network_connect 1
```

## logspout

```shell
docker service create --name logspout \
  --mode global \
  --mount type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  gliderlabs/logspout syslog+tcp://10.1.2.4:514
```

## network

```shell
docker network create --driver overlay --subnet 192.168.1.0/24  elknet
```

### logstash

1. 添加日志解析配置 **/data1/logstash/service.conf**

```shell
input {
  syslog {
    port => '10514'
  }
}
filter {
  dissect {
     mapping => { 
       "message" => "%{?prefix} %{log_time} %{} %{service_name}.%{container_no}.%{container_id} %{?pid} - - %{log_content}"
     }
  }
  date {
    match => ["log_time", "ISO8601"]
  }
  # 判断日志类型
  if [service_name] =~ /^app+/ {
    # todo php日志识别
    mutate {
      add_field => ["log_type", "app"]
    }
  } else if [service_name] =~ /^http+/ {
    mutate {
      add_field => ["log_type", "nginx"]
    }
  } else {
    mutate {
      add_field => ["log_type", "other"]
    }
  }
  # 解析json 格式日志
  if [log_type] in ["app", "nginx"]{
    json {
      source => "log_content"
    }
  } 
  # 获取落地路径
  if [log_type] == "other" {
    mutate {
      add_field => ["log_path", "%{service_name}/%{+YYYYMMdd}.log"]
    }
  }
  # 去掉不必要的数据项
  mutate {
    remove_field => ["message"]
  }
}
output {
  file {
    path => "/mnt/logs/%{log_type}/%{log_path}"
    gzip => false #按gzip方式压缩
    flush_interval => 2  #指定刷入间隔(秒数)，0代表实时写入 默认2s
    codec => line {
      format => "%{log_content}"
    }
  }
  if [log_type] in ["app", "nginx"]{
    elasticsearch {
      hosts => ["10.1.2.4:9200"]
      index => "logstash-%{log_type}-%{app}-%{+YYYYMM}"
      document_type => "%{level}"
      flush_size => 1000 #批量上送数据最大条数 不会超过pipeline.batch.size 默认500
      idle_flush_time => 1 #批量上送数据间隔 默认1s
    }
  } 
}
```

```shell
input {
  stdin {}
}   
output {
   stdout { codec => rubydebug }
}
```

```shell
docker config create logstash-service /data1/logstash/service.conf
```

2. 启动

```shell
docker service create --name logstash \
    --network=elknet \
    --with-registry-auth \
    -p 514:10514 \
    --replicas 2 \
    --limit-cpu 1 \
    --limit-memory 2048mb \
    --config source=logstash-app,target=/conf.d/app.conf \
    dockerhub.jrmf360.com/log/logstash-with-dissect:latest
```

## ElasticSearch

```shell
sudo sysctl -w vm.max_map_count=262144
```

```shell
mkdir /tmp/es
chmod 777 -R /tmp/es
```

```shell
docker service create --name elasticsearch --network=elknet \
  --mode global \
  --with-registry-auth \
  --mount type=bind,src=/tmp/es,dst=/usr/share/elasticsearch/data \
  --env SERVICE_NAME=es \
  --publish 9200:9200 \
  --env discovery.zen.minimum_master_nodes=3 \
  --env path.data=/usr/share/elasticsearch/data  \
  dockerhub.jrmf360.com/log/swarm-elasticsearch:latest
```

## Kibana

```shell
docker service create --name kibana --network=elknet \
  --replicas 2 \
  --publish 5601:5601 \
  --env  ELASTICSEARCH_URL=http://es:9200 \
  kibana
```

## 统一部署

**compose-stack-log.yml**

```yaml
version: "3.3"

services:
  logstash:
    image: logstash:5.4.0
    ports:
      - 514:10514
    networks:
      - elknet
    configs:
      - source: logstash-service
        target: /conf.d/service.conf
    volumes:
      - type: bind
        source: /tmp/logs
        target: /mnt/logs
    deploy:
      replicas: 2
    command: ["-f", "/conf.d"]
  elasticsearch:
    image: ifintech/swarm-elasticsearch
    ports:
      - 9200:9200
    networks:
      - elknet
    environment:
      SERVICE_NAME: log_elasticsearch
      discovery.zen.minimum_master_nodes: 3
      path.data: /usr/share/elasticsearch/data
    volumes:
      - type: bind
        source: /tmp/es
        target: /usr/share/elasticsearch/data
    deploy:
      mode: global
  kibana:
    image: kibana
    ports:
      - 5601:5601
    networks:
      - elknet
    environment:
      ELASTICSEARCH_URL: http://log_elasticsearch:9200
    deploy:
      replicas: 2
configs:
  logstash-service:
    file: /data1/logstash/service.conf
networks:
  elknet:
    external: true
```

```shell
docker stack deploy log --compose-file compose-stack-log.yml
```

```shell
docker service create --name wallet \
  --with-registry-auth \
  --replicas 2 \
  --env PHP_RUN_ENV=product \
  -p 30000:9000 \
  dockerhub.jrmf360.com/cash-loan/wallet php-fpm
```

