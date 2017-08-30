```yaml
version: "3.2"

services:
  static: # web前端应用
      image: {IMAGE_ADDRESS}
      networks:
        - servicenet
      deploy:
        replicas: 2
        resources:
          limits:
            cpus: '0.5'
            memory: 256M
          reservations:
            cpus: '0.1'
            memory: 100M
        update_config:
          parallelism: 1
          delay: 10s
  java:
    image: {IMAGE_ADDRESS}
    networks:
      - servicenet
    environment:
      RUN_ENV: product
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 500M
      update_config:
        parallelism: 1
        delay: 30s
networks:
  servicenet:
    external: true
```

