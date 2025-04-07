# Микросервисная архитектура с использованием 2 сервисов (расчет кредитного платежа и получение информации о кредитных программах)
# Docker-compose
```yml
version: '3.8'

services:
  db_credit_programs: # создаем контейнер под БД credit-programs
    image: postgres:latest
    container_name: db_credit_programs
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: credit_programs
    ports:
      - "5435:5432"
    networks:
      - monitoring
    volumes:
      - ./db_credit_programs_data:/var/lib/postgresql/data

  db_credit_loans: # создаем контейнер под БД credit-loans
    image: postgres:latest
    container_name: db_credit_loans
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: credit_loans
    ports:
      - "5437:5432"
    networks:
      - monitoring
    volumes:
      - ./db_credit_loans_data:/var/lib/postgresql/data

  credit-programs: # создаем контейнер под "сервис/микросервис" для его связки с другими системами + работы с метриками
    image: credit-programs-image
    container_name: credit-programs
    ports:
      - "8083:8080"
    networks:
      - monitoring
    depends_on:
      - db_credit_programs

  credit-loans: # создаем контейнер под "сервис/микросервис" для его связки с другими системами + работы с метриками
    image: credit-loans-image
    container_name: credit-loans
    ports:
      - "8082:8080"
    networks:
      - monitoring
    depends_on:
      - db_credit_loans

  nginx: # выступает в качестве шлюза между пользаком и сервером
    image: nginx:latest
    container_name: nginx_gateway
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - monitoring
    depends_on:
      - credit-programs
      - credit-loans

  cadvisor: #собирает метрики с контейнеров
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    container_name: cadvisor
    ports:
      - "8087:8080"
    volumes:
      - "/:/rootfs:ro"
      - "/var/run:/var/run:ro"
      - "/sys:/sys:ro"
      - "/var/lib/docker/:/var/lib/docker:ro"
    networks:
      - monitoring
    depends_on:
      - credit-programs
      - credit-loans
      - db_credit_programs
      - db_credit_loans

  node-exporter: # агент сбора метрик
    image: prom/node-exporter:v1.6.1
    container_name: node-exporter
    ports:
      - "9100:9100"
    networks:
      - monitoring
    depends_on:
      - cadvisor

  prometheus: #собирает метрики с cAdvisor и обрабатывает их
    image: prom/prometheus:v2.45.0
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    networks:
      - monitoring
    depends_on:
      - cadvisor
      - node-exporter

  grafana: # визуализирует полученные метрики
    image: grafana/grafana:10.0.0
    container_name: grafana
    ports:
      - "3000:3000"
    networks:
      - monitoring
    depends_on:
      - prometheus
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - ./grafana-storage:/var/lib/grafana

networks:
  monitoring:
    driver: bridge

volumes:
  db_credit_programs_data:
  db_credit_loans_data:

```
## prometheus.yml для связки с cAdvisor
!расписать поподробнее!
```yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['172.18.0.7:8087']  #либо вместо ip-address можно просто localhost
```
## Какие могут быть проблемы после развертки всех систем в контейнеры
1. f
2. f
3. f
4. f
5. f
# cAdvisor
fg
# Nginx

# Node-exporter

# Prometheus

# Grafana









```yml
version: '3.8'

services:
  # ======================== ETCD ========================
  etcd:
    image: quay.io/coreos/etcd:v3.5.0
    container_name: etcd
    environment:
      - ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379
      - ETCD_ADVERTISE_CLIENT_URLS=http://etcd:2379
    networks:
      - monitoring
    ports:
      - "2379:2379"

  # ======================== Patroni Cluster ========================
  patroni1:
    image: zalando/patroni
    container_name: patroni1
    environment:
      - PATRONI_NAME=postgres1
      - PATRONI_RESTAPI_LISTEN=0.0.0.0:8008
      - PATRONI_ETCD_HOSTS=etcd:2379
      - PATRONI_SCOPE=postgres-cluster
      - PATRONI_POSTGRESQL_DATA_DIR=/home/postgres/pgdata
      - PATRONI_POSTGRESQL_CONNECT_ADDRESS=patroni1:5432
      - PATRONI_RESTAPI_CONNECT_ADDRESS=patroni1:8008
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - PATRONI_POSTGRESQL_PASSWORD=password
      - PATRONI_POSTGRESQL_USERNAME=user
    ports:
      - "5432:5432"
    networks:
      - monitoring
    depends_on:
      - etcd

  patroni2:
    image: zalando/patroni
    container_name: patroni2
    environment:
      - PATRONI_NAME=postgres2
      - PATRONI_RESTAPI_LISTEN=0.0.0.0:8008
      - PATRONI_ETCD_HOSTS=etcd:2379
      - PATRONI_SCOPE=postgres-cluster
      - PATRONI_POSTGRESQL_DATA_DIR=/home/postgres/pgdata
      - PATRONI_POSTGRESQL_CONNECT_ADDRESS=patroni2:5432
      - PATRONI_RESTAPI_CONNECT_ADDRESS=patroni2:8008
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
      - PATRONI_POSTGRESQL_PASSWORD=password
      - PATRONI_POSTGRESQL_USERNAME=user
    ports:
      - "5433:5432"
    networks:
      - monitoring
    depends_on:
      - etcd

  haproxy:
    image: haproxy:2.7
    container_name: haproxy
    ports:
      - "5434:5432"
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
    networks:
      - monitoring
    depends_on:
      - patroni1
      - patroni2

  # ======================== Сервисы ========================

  credit-programs:
    image: credit-programs-image
    container_name: credit-programs
    restart: always
    ports:
      - "8083:8080"
    networks:
      - monitoring
    environment:
      - POSTGRES_HOST=haproxy
      - POSTGRES_PORT=5432
      - POSTGRES_DB=credit_programs
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
    depends_on:
      - haproxy

  credit-loans:
    image: credit-loans-image
    container_name: credit-loans
    restart: always
    ports:
      - "8082:8080"
    networks:
      - monitoring
    environment:
      - POSTGRES_HOST=haproxy
      - POSTGRES_PORT=5432
      - POSTGRES_DB=credit_loans
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
    depends_on:
      - haproxy

  nginx:
    image: nginx:latest
    container_name: nginx_gateway
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - monitoring
    depends_on:
      - credit-programs
      - credit-loans

  # ======================== Мониторинг ========================

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    container_name: cadvisor
    restart: always
    ports:
      - "8087:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:v1.6.1
    container_name: node-exporter
    restart: always
    ports:
      - "9100:9100"
    networks:
      - monitoring

  prometheus:
    image: prom/prometheus:v2.45.0
    container_name: prometheus
    restart: always
    ports:
      - "9090:9090"
    command:
      - --config.file=/etc/prometheus/prometheus.yml
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    networks:
      - monitoring
    depends_on:
      - cadvisor
      - node-exporter
      - alertmanager

  grafana:
    image: grafana/grafana:10.0.0
    container_name: grafana
    restart: always
    ports:
      - "3000:3000"
    networks:
      - monitoring
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - ./grafana-storage:/var/lib/grafana

  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    restart: always
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge

volumes:
  grafana-storage:

```
