# Домашнее задание к занятию «ELK»

### Выполнил: Шушнов Александр

---

## Стенд

| Параметр | Значение |
| --- | --- |
| ОС | Ubuntu 24.04.4 LTS (VPS, 2 vCPU / 2 ГБ RAM + 4 ГБ swap) |
| Elasticsearch | 7.17.29 |
| Kibana | 7.17.29 |
| Logstash | 7.17.29 |
| Filebeat | 7.17.29 |
| Nginx | 1.24.0 (порт 8081 — 80 занят HAProxy с прошлого ДЗ) |

Все компоненты установлены из официального apt-репозитория Elastic (`https://artifacts.elastic.co/packages/7.x/apt`).
Так как памяти на сервере немного, heap Elasticsearch ограничен 512 МБ (`/etc/elasticsearch/jvm.options.d/heap.options`).

---

## Задание 1. Elasticsearch

Конфиг `/etc/elasticsearch/elasticsearch.yml`:

```yaml
cluster.name: shushnov-elk-7f3k
node.name: us-det01
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 127.0.0.1
http.port: 9200
discovery.type: single-node
xpack.security.enabled: false
```

```bash
systemctl enable --now elasticsearch
curl -X GET 'localhost:9200/_cluster/health?pretty'
```

![cluster health](img/01-es-cluster-health.png)

---

## Задание 2. Kibana

Конфиг `/etc/kibana/kibana.yml`:

```yaml
server.port: 5601
server.host: "0.0.0.0"
server.name: "us-det01"
elasticsearch.hosts: ["http://127.0.0.1:9200"]
```

Запрос `GET /_cluster/health?pretty` в консоли `http://66.187.7.57:5601/app/dev_tools#/console`:

![Kibana Dev Tools](img/02-kibana-dev-tools.jpg)

---

## Задание 3. Logstash

Nginx пишет access-лог в `/var/log/nginx/hw-access.log` (директива `access_log` в `/etc/nginx/sites-available/default`).
Пользователь `logstash` добавлен в группу `adm`, чтобы читать логи nginx: `usermod -aG adm logstash`.

Пайплайн `/etc/logstash/conf.d/nginx.conf` ([config/logstash-nginx.conf](config/logstash-nginx.conf)) — читает лог, разбирает его
grok-шаблоном `COMBINEDAPACHELOG`, берёт время из лога и отправляет в индекс `nginx-access-logstash-*`:

```ruby
input {
  file {
    path => "/var/log/nginx/hw-access.log"
    start_position => "beginning"
    sincedb_path => "/var/lib/logstash/sincedb_nginx_hw"
  }
}

filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }
  date {
    match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
  }
  mutate {
    add_field => { "shipper" => "logstash" }
  }
}

output {
  elasticsearch {
    hosts => ["http://127.0.0.1:9200"]
    index => "nginx-access-logstash-%{+YYYY.MM.dd}"
  }
}
```

Логи Nginx в Kibana (Discover, index pattern `nginx-access-logstash-*`):

![nginx logs via logstash](img/03-kibana-nginx-logstash.png)

---

## Задание 4. Filebeat

Logstash остановлен (`systemctl disable --now logstash`), поставка логов nginx переключена на Filebeat
с модулем `nginx` — разбор строк лога делает ingest pipeline модуля прямо в Elasticsearch.

`/etc/filebeat/filebeat.yml` ([config/filebeat.yml](config/filebeat.yml)):

```yaml
filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false

fields:
  shipper: filebeat
fields_under_root: true

setup.kibana:
  host: "127.0.0.1:5601"

output.elasticsearch:
  hosts: ["127.0.0.1:9200"]
```

`/etc/filebeat/modules.d/nginx.yml` ([config/filebeat-nginx-module.yml](config/filebeat-nginx-module.yml)):

```yaml
- module: nginx
  access:
    enabled: true
    var.paths: ["/var/log/nginx/hw-access.log"]
  error:
    enabled: false
  ingress_controller:
    enabled: false
```

```bash
filebeat modules enable nginx
filebeat test config
filebeat setup --index-management --pipelines --modules nginx
systemctl enable --now filebeat
```

Логи Nginx, отправленные через Filebeat (index pattern `filebeat-*`, поля `agent.type` и `shipper` = `filebeat`):

![nginx logs via filebeat](img/04-kibana-nginx-filebeat.png)
