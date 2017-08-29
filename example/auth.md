# compose-stack-auth.yml



```yaml
version: "3.2"

services:
  ldap:
    image: ifintech/ldap2http
    ports:
      - "10389:10389"
    environment:
      HOST: 0.0.0.0
      PORT: 10389
      AUTH_URL: http://auth_nginx
      AUTH_TOKEN: {AUTH_SECRET}
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.1'
          memory: 100M
      update_config:
        parallelism: 1
        delay: 30s
    networks:
      - servicenet
  php:
    image: dockerhub.jrmf360.com/sa/auth
    command: php-fpm
    environment:
      RUN_ENV: product
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1'
          memory: 512M
        reservations:
          cpus: '0.1'
          memory: 100M
      update_config:
        parallelism: 1
        delay: 30s
    networks:
      - servicenet
  nginx:
    image: ifintech/nginx-php
    networks:
      - servicenet
    environment:
      APP_NAME: auth
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

