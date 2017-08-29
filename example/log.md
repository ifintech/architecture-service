# compose-stack-log.yml

```yaml
   version: "3.2"

   services:
     logstash:
       image: ifintech/logstash
       networks:
         - servicenet
       volumes:
         - type: bind
           source: /data1/logs #日志落地路径
           target: /data1/logs
       deploy:
         replicas: 1
         resources:
           limits:
             cpus: '2'
             memory: 2G
           reservations:
             cpus: '.1'
             memory: 100M
         update_config:
           parallelism: 1
           delay: 30s
     elasticsearch:
       image: ifintech/swarm-elasticsearch
       ports:
         - 9200:9200
       networks:
         - servicenet
       environment:
         SERVICE_NAME: log_elasticsearch
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
             - node.labels.type == es
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
         ELASTICSEARCH_URL: http://log_elasticsearch:9200
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

