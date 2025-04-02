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
