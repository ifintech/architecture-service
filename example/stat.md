# compose-stack-stat.yml


```yaml
version: "3.2"

services:
  es:
    image: ifintech/swarm-elasticsearch
    ports:
      - 9201:9200
    networks:
      - servicenet
    environment:
      SERVICE_NAME: stat_es
      discovery.zen.minimum_master_nodes: 3
      path.data: /usr/share/elasticsearch/data
    volumes:
      - type: bind
        source: /mnt/docker/es #es数据存储路径
        target: /usr/share/elasticsearch/data
    deploy:
      replicas: 3
      placement:
        constraints:
          - node.labels.type == stat-es
      resources:
        limits:
          cpus: '1'
          memory: 4G
        reservations:
          cpus: '.1'
          memory: 1G
      update_config:
        parallelism: 1
        delay: 30s
  kibana:
    image: kibana:5.5.0
    networks:
      - servicenet
    environment:
      ELASTICSEARCH_URL: http://stat_es:9200
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '.5'
          memory: 1G
        reservations:
          cpus: '.1'
          memory: 50M
      update_config:
        parallelism: 1
        delay: 10s
networks:
  servicenet:
    external: true
```