# Домашнее задание к занятию 5. «Elasticsearch» - Дрибноход Давид

## Задача 1

Dockerfile

Предварительно скачал архивы используя прокси и разместил в папке рядом с dockerfile

```docker
FROM centos:7
COPY elasticsearch-8.10.2-linux-x86_64.tar.gz  /opt
COPY elasticsearch-8.10.2-linux-x86_64.tar.gz.sha512  /opt
RUN cd /opt && \
    groupadd elasticsearch && \
    useradd -c "elasticsearch" -g elasticsearch elasticsearch &&\
    yum update -y && yum -y install wget perl-Digest-SHA && \
    shasum -a 512 -c elasticsearch-8.10.2-linux-x86_64.tar.gz.sha512 && \
    tar -xzf elasticsearch-8.10.2-linux-x86_64.tar.gz && \
    rm elasticsearch-8.10.2-linux-x86_64.tar.gz elasticsearch-8.10.2-linux-x86_64.tar.gz.sha512 && \ 
    mkdir /var/lib/data && chmod -R 777 /var/lib/data && \
    chown -R elasticsearch:elasticsearch /opt/elasticsearch-8.10.2 && \
    yum -y remove wget perl-Digest-SHA && \
    yum clean all
USER elasticsearch
WORKDIR /opt/elasticsearch-8.10.2/
COPY elasticsearch.yml  config/
EXPOSE 9200 9300
ENTRYPOINT ["bin/elasticsearch"]
```

Файл elasticsearch.yml

```yml
node:
  name: netology_test
path:
  data: /var/lib/data
xpack.ml.enabled: false
```
ссылка на образ

[https://hub.docker.com/repository/docker/drdavidn/elasticsearch/](https://hub.docker.com/repository/docker/drdavidn/elasticsearch/)

```bash
sh-4.2$ curl --insecure -u elastic:elastic https://localhost:9200
{
  "name" : "netology_test",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "bqxqABFNRLq4iHzN-1GZfQ",
  "version" : {
    "number" : "8.10.2",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "6d20dd8ce62365be9b1aca96427de4622e970e9e",
    "build_date" : "2023-09-19T08:16:24.564900370Z",
    "build_snapshot" : false,
    "lucene_version" : "9.7.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

## Задача 2

Ознакомтесь с [документацией](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-create-index.html) 
и добавьте в `elasticsearch` 3 индекса, в соответствии со таблицей:

| Имя | Количество реплик | Количество шард |
|-----|-------------------|-----------------|
| ind-1| 0 | 1 |
| ind-2 | 1 | 2 |
| ind-3 | 2 | 4 |

```bash
sh-4.2$ curl -X PUT --insecure -u elastic:elastic "https://localhost:9200/ind-1?pretty" -H 'Content-Type: application/json' -d'
{
  "settings": {
    "index": {
      "number_of_shards": 1,  
      "number_of_replicas": 0 
    }
  }
}
'

sh-4.2$ curl -X PUT --insecure -u elastic:elastic "https://localhost:9200/ind-2?pretty" -H 'Content-Type: application/json' -d'
{
  "settings": {
    "index": {
      "number_of_shards": 2,  
      "number_of_replicas": 1 
    }
  }
}
'

sh-4.2$ curl -X PUT --insecure -u elastic:elastic "https://localhost:9200/ind-3?pretty" -H 'Content-Type: application/json' -d'
{
  "settings": {
    "index": {
      "number_of_shards": 4,  
      "number_of_replicas": 2 
    }
  }
}
'
```

Получите список индексов и их статусов, используя API и **приведите в ответе** на задание.

```bash
sh-4.2$ curl -X GET --insecure -u elastic:elastic "https://localhost:9200/_cat/indices?v=true"
health status index uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   ind-1 GigfKvNjRTiH0Ur45FW1vA   1   0          0            0       226b           226b
yellow open   ind-3 Xsnv9KcsSWC9n0KU3UQzgg   4   2          0            0       904b           904b
yellow open   ind-2 doPJzQeKRrSQOQl_InnMZg   2   1          0            0       452b           452b
```

Получите состояние кластера `elasticsearch`, используя API.

```bash
sh-4.2$ curl -X GET --insecure -u elastic:elastic "https://localhost:9200/_cluster/health?pretty"
{
  "cluster_name" : "elasticsearch",
  "status" : "yellow",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 8,
  "active_shards" : 8,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 10,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 44.44444444444444
}
```
Индексы и кластер находятся в yellow, так как при создании индексов мы указали количество реплик больше 1. В кластере у нас 1 нода, поэтому реплицировать индексы некуда.

Удалите все индексы.

```bash
sh-4.2$ curl -X DELETE --insecure -u elastic:elastic "https://localhost:9200/ind-1?pretty"
{
  "acknowledged" : true
}
sh-4.2$ curl -X DELETE --insecure -u elastic:elastic "https://localhost:9200/ind-2?pretty"
{
  "acknowledged" : true
}
sh-4.2$ curl -X DELETE --insecure -u elastic:elastic "https://localhost:9200/ind-3?pretty"
{
  "acknowledged" : true
}
```

## Задача 3

В данном задании вы научитесь:
- создавать бэкапы данных
- восстанавливать индексы из бэкапов

Создали директорию `/opt/elasticsearch-8.10.2/snapshots`. В конфигурационный файл `elasticsearch.yml` внесли параметр `path.repo: ["/opt/elasticsearch-8.10.2/snapshots"]`

Выполнили запрос на регистрацию `snapshot` репозитория

```bash
sh-4.2$ mkdir /opt/elasticsearch-8.10.2/snapshots
sh-4.2$ cd /opt/elasticsearch-8.10.2/snapshots
sh-4.2$ curl -X PUT --insecure -u elastic:elastic "https://localhost:9200/_snapshot/netology_backup?pretty" -H 'Content-Type: application/json' -d'
",
  "settings": {
    "location": "/opt/elasticsearch-8.10.2/snapshots"
  }
}
'
> {
>   "type": "fs",
>   "settings": {
>     "location": "/opt/elasticsearch-8.10.2/snapshots"
>   }
> }
> '

{
  "acknowledged" : true
}
```

Создали индекс `test` с 0 реплик и 1 шардом

```bash
sh-4.2$ curl -X GET --insecure -u elastic:elastic "https://localhost:9200/_cat/indices?v=true"
health status index uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   test  Clhz_TRpB6e1E7_skjOu_v   1   0          0            0       225b           225b
```
Сделали snapshot кластера

```bash
sh-4.2$ curl -X PUT --insecure -u elastic:elastic "https://localhost:9200/_snapshot/netology_backup/%3Cmy_snapshot_%7Bnow%2Fd%7D%3E?pretty"
{
  "accepted" : true
}
```
Список файлов в директории со `snapshot`ами

```bash
sh-4.2$ ls -l
total 36
-rw-r--r-- 1 elasticsearch elasticsearch  1107 Oct 5 06:12 index-0
-rw-r--r-- 1 elasticsearch elasticsearch     8 Oct 5 06:12 index.latest
drwxr-xr-x 5 elasticsearch elasticsearch  4096 Oct 5 06:12 indices
-rw-r--r-- 1 elasticsearch elasticsearch 16595 Oct 5 06:12 meta-y1W7p4kFTO2S7PM8rZ75AQ.dat
-rw-r--r-- 1 elasticsearch elasticsearch   401 Oct 5 06:12 snap-y1W7p4kFTO2S7PM8rZ75AQ.dat
```

Удалили индекс `test` и создали индекс `test-2`.

```bash
sh-4.2$ curl -X GET --insecure -u elastic:elastic "https://localhost:9200/_cat/indices?v=true"
health status index uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   test2 4dslkULPSiJdXyGvCVrdDG   1   0          0            0       225b           225b
```

Получили список доступных `snapshot`ов

```bash
sh-4.2$ curl -X GET --insecure -u elastic:elastic "https://localhost:9200/_snapshot/netology_backup/*?verbose=false&pretty"
{
  "snapshots" : [
    {
      "snapshot" : "my_snapshot_2023.10.05",
      "uuid" : "y1W7p4kFTO2S7PM8rZ75AQ",
      "repository" : "netology_backup",
      "indices" : [
        ".geoip_databases",
        ".security-7",
        "test"
      ],
      "data_streams" : [ ],
      "state" : "SUCCESS"
    }
  ],
  "total" : 1,
  "remaining" : 0
}
```

Восстановили состояние кластера `elasticsearch` из `snapshot`, созданного ранее.

```bash
sh-4.2$ curl -X POST --insecure -u elastic:elastic "https://localhost:9200/_snapshot/netology_backup/my_snapshot_2023.10.05/_restore?pretty" -H 'Content-Type: application/json' -d'
> {
>   "indices": "*",
>   "include_global_state": true
> }
> '

{
  "accepted" : true
}
```
Итоговый список индексов

```bash
sh-4.2$ curl -X GET --insecure -u elastic:elastic "https://localhost:9200/_cat/indices?v=true"
health status index uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   test2 4dslkULPSiJdXyGvCVrdDG   1   0          0            0       225b           225b
green  open   test  Clhz_TRpB6e1E7_skjOu_v   1   0          0            0       225b           225b
```
---
