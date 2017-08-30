# compose-stack-service.yml 

```yml
version: "3.2"

services:
  php:
    image: ifintech/service
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
        delay: 30s
    command: ["php-fpm"]
  nginx:
    image: ifintech/nginx-php
    networks:
      - servicenet
    environment:
      APP_NAME: service
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
        delay: 30s
networks:
  servicenet:
    external: true
    
```