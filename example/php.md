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
  php: # php应用
    image: {IMAGE_ADDRESS}
    networks:
      - servicenet
    environment:
      RUN_ENV: product
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1'
          memory: 1G
        reservations:
          cpus: '0.1'
          memory: 100M
      update_config:
        parallelism: 1
        delay: 10s
    command: ["php-fpm"]
  nginx: #nginx应用 负责给php应用提供http通信的支持
    image: ifintech/nginx-php
    networks:
      - servicenet
    environment:
      APP_NAME: {APP_NAME}
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
        reservations:
          cpus: '0.1'
          memory: 50M
      update_config:
        parallelism: 1
        delay: 10s
networks:
  servicenet:
    external: true
```

