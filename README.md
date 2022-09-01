# Домашнее задание к занятию "6.5. Elasticsearch" - dev-17_subd_6.5.-yakovlev_vs
Elasticsearch

## Задача 1

В этом задании вы потренируетесь в:
- установке elasticsearch
- первоначальном конфигурировании elastcisearch
- запуске elasticsearch в docker

Используя докер образ [centos:7](https://hub.docker.com/_/centos) как базовый и 
[документацию по установке и запуску Elastcisearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/targz.html):

- составьте Dockerfile-манифест для elasticsearch
- соберите docker-образ и сделайте `push` в ваш docker.io репозиторий
- запустите контейнер из получившегося образа и выполните запрос пути `/` c хост-машины

Требования к `elasticsearch.yml`:
- данные `path` должны сохраняться в `/var/lib`
- имя ноды должно быть `netology_test`

В ответе приведите:
- текст Dockerfile манифеста
- ссылку на образ в репозитории dockerhub
- ответ `elasticsearch` на запрос пути `/` в json виде

Подсказки:
- возможно вам понадобится установка пакета perl-Digest-SHA для корректной работы пакета shasum
- при сетевых проблемах внимательно изучите кластерные и сетевые настройки в elasticsearch.yml
- при некоторых проблемах вам поможет docker директива ulimit
- elasticsearch в логах обычно описывает проблему и пути ее решения

Далее мы будем работать с данным экземпляром elasticsearch.

#### Решение

- текст Dockerfile манифеста
```dockerfile
# Манифест Docker образа.
FROM centos:7

EXPOSE 9200 9300

ENV PATH=/usr/lib:/usr/lib/jvm/jre-11/bin:$PATH

RUN yum install java-11-openjdk -y 

COPY elasticsearch-8.3.3-linux-x86_64.tar.gz /
COPY elasticsearch-8.3.3-linux-x86_64.tar.gz.sha512 /

RUN yum install -y perl-Digest-SHA && \
    yum -y install wget && \
    sha512sum -c elasticsearch-8.3.3-linux-x86_64.tar.gz.sha512 && \
    tar -xzf elasticsearch-8.3.3-linux-x86_64.tar.gz && \
    rm elasticsearch-8.3.3-linux-x86_64.tar.gz && \
    rm elasticsearch-8.3.3-linux-x86_64.tar.gz.sha512
    
COPY elasticsearch.yml /elasticsearch-8.3.3/config/

ENV ES_JAVA_HOME=/elasticsearch-8.3.3/jdk/
ENV ES_HOME=/elasticsearch-8.3.3

RUN groupadd elasticsearch \
    && useradd -g elasticsearch elasticsearch
    
RUN mkdir /var/lib/logs \
    && chown elasticsearch:elasticsearch /var/lib/logs \
    && mkdir /var/lib/data \
    && chown elasticsearch:elasticsearch /var/lib/data \
    && chown -R elasticsearch:elasticsearch /elasticsearch-8.3.3/
RUN mkdir /elasticsearch-8.3.3/snapshots &&\
    chown elasticsearch:elasticsearch /elasticsearch-8.3.3/snapshots
    
USER elasticsearch

CMD ["/usr/sbin/init"]
CMD ["/elasticsearch-8.3.3/bin/elasticsearch"]
```

- ссылка на образ в репозитории dockerhub
```bash
docker pull valdem88/elasticsearch:v5
```
- ответ `elasticsearch` на запрос пути `/` в json виде
```bash
root@server1:~/el01# docker run --rm -d --name el05 -p 9200:9200 -p 9300:9300 valdem88/elasticsearch:v5
Passwordroot@server1:~/el01# docker ps
CONTAINER ID   IMAGE                       COMMAND                  CREATED             STATUS             PORTS                                                                                  NAMES
32ad9e268a83   valdem88/elasticsearch:v5   "/elasticsearch-8.3.…"   About an hour ago   Up About an hour   0.0.0.0:9200->9200/tcp, :::9200->9200/tcp, 0.0.0.0:9300->9300/tcp, :::9300->9300/tcp   el05
dce220a02abb   postgres:12                 "docker-entrypoint.s…"   5 weeks ago         Up 26 hours        0.0.0.0:5432->5432/tcp 
```
```json
root@server1:~/el01# curl -X GET 'http://localhost:9200/'
{
  "name" : "32ad9e268a83",
  "cluster_name" : "netology_test",
  "cluster_uuid" : "lX4flkGwRAK3TcYMbjVyOA",
  "version" : {
    "number" : "8.3.3",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "801fed82df74dbe537f89b71b098ccaff88d2c56",
    "build_date" : "2022-07-23T19:30:09.227964828Z",
    "build_snapshot" : false,
    "lucene_version" : "9.2.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

## Задача 2

В этом задании вы научитесь:
- создавать и удалять индексы
- изучать состояние кластера
- обосновывать причину деградации доступности данных

Ознакомтесь с [документацией](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-create-index.html) 
и добавьте в `elasticsearch` 3 индекса, в соответствии со таблицей:

| Имя | Количество реплик | Количество шард |
|-----|-------------------|-----------------|
| ind-1| 0 | 1 |
| ind-2 | 1 | 2 |
| ind-3 | 2 | 4 |

Получите список индексов и их статусов, используя API и **приведите в ответе** на задание.

Получите состояние кластера `elasticsearch`, используя API.

Как вы думаете, почему часть индексов и кластер находится в состоянии yellow?

Удалите все индексы.

**Важно**

При проектировании кластера elasticsearch нужно корректно рассчитывать количество реплик и шард,
иначе возможна потеря данных индексов, вплоть до полной, при деградации системы.

#### Решение 

- создаю индексы
```bash
curl -X PUT localhost:9200/ind-1 -H 'Content-Type: application/json' -d'{ "settings": { "number_of_shards": 1,  "number_of_replicas": 0 }}'
curl -X PUT localhost:9200/ind-2 -H 'Content-Type: application/json' -d'{ "settings": { "number_of_shards": 2,  "number_of_replicas": 1 }}'
curl -X PUT localhost:9200/ind-3 -H 'Content-Type: application/json' -d'{ "settings": { "number_of_shards": 4,  "number_of_replicas": 2 }}' 
```
- список индексов и их статусов
```bash
root@server1:~/el01# curl -X GET 'http://localhost:9200/_cat/indices?v'
health status index uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   ind-2 FnyzfOGaSSOdZZdQrtf_ug   2   1          0            0       450b           450b
green  open   ind-1 _OpX1JhjQhK_qFu-zHU60g   1   0          0            0       225b           225b
yellow open   ind-3 Wa7FSYidSoG6FTd2vnwqaQ   4   2          0            0       900b           900b
```
- состояние кластера `elasticsearch`
```bash
root@server1:~/el01# curl -X GET "localhost:9200/_cluster/health?pretty"
{
  "cluster_name" : "netology_test",
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
- Как вы думаете, почему часть индексов и кластер находится в состоянии yellow?

`Ответ` - Первичный шард и реплика не могут находиться на одном узле, если копия не назначена. Один узел не может размещать копии

- удаляю индексы

```bash
root@server1:~/el01# curl -X DELETE 'http://localhost:9200/ind-1?pretty'
{
  "acknowledged" : true
}
root@server1:~/el01# curl -X DELETE 'http://localhost:9200/ind-2?pretty'
{
  "acknowledged" : true
}
root@server1:~/el01# curl -X DELETE 'http://localhost:9200/ind-3?pretty'
{
  "acknowledged" : true
}
root@server1:~/el01# curl -X GET 'http://localhost:9200/_cat/indices?v'
health status index uuid pri rep docs.count docs.deleted store.size pri.store.size
root@server1:~/el01#
```

## Задача 3

В данном задании вы научитесь:
- создавать бэкапы данных
- восстанавливать индексы из бэкапов

Создайте директорию `{путь до корневой директории с elasticsearch в образе}/snapshots`.

Используя API [зарегистрируйте](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshots-register-repository.html#snapshots-register-repository) 
данную директорию как `snapshot repository` c именем `netology_backup`.

**Приведите в ответе** запрос API и результат вызова API для создания репозитория.

Создайте индекс `test` с 0 реплик и 1 шардом и **приведите в ответе** список индексов.

[Создайте `snapshot`](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshots-take-snapshot.html) 
состояния кластера `elasticsearch`.

**Приведите в ответе** список файлов в директории со `snapshot`ами.

Удалите индекс `test` и создайте индекс `test-2`. **Приведите в ответе** список индексов.

[Восстановите](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshots-restore-snapshot.html) состояние
кластера `elasticsearch` из `snapshot`, созданного ранее. 

**Приведите в ответе** запрос к API восстановления и итоговый список индексов.

Подсказки:
- возможно вам понадобится доработать `elasticsearch.yml` в части директивы `path.repo` и перезапустить `elasticsearch`

#### Решение

- Создание директории для `snapshot`
```bash
root@server1:~/el01# curl -X PUT "localhost:9200/_snapshot/netology_backup?pretty" -H 'Content-Type: application/json' -d'
> {
>   "type": "fs",
>   "settings": {
>     "location": "/elasticsearch-8.3.3/snapshots"
>   }
> }'
{
  "acknowledged" : true
}
```
- Создайте индекс `test` с 0 реплик и 1 шардом и **приведите в ответе** список индексов.
```bash
root@server1:~/el01# curl -X PUT localhost:9200/test -H 'Content-Type: application/json' -d'{ "settings": { "number_of_shards": 1,  "number_of_replicas": 0 }}'
{"acknowledged":true,"shards_acknowledged":true,"index":"test"}root@server1:~/el01#
root@server1:~/el01# curl -X GET 'http://localhost:9200/test?pretty'
{
  "test" : {
    "aliases" : { },
    "mappings" : { },
    "settings" : {
      "index" : {
        "routing" : {
          "allocation" : {
            "include" : {
              "_tier_preference" : "data_content"
            }
          }
        },
        "number_of_shards" : "1",
        "provided_name" : "test",
        "creation_date" : "1662065738336",
        "number_of_replicas" : "0",
        "uuid" : "8ETmzTwzR8meMaIbhlRf4w",
        "version" : {
          "created" : "8030399"
        }
      }
    }
  }
}
```
- [Создайте `snapshot`] **Приведите в ответе** список файлов в директории со `snapshot`ами.

```bash
root@server1:~/el01# curl -X PUT "localhost:9200/_snapshot/netology_backup/snapshot_1?wait_for_completion=true&pretty"
{
  "snapshot" : {
    "snapshot" : "snapshot_1",
    "uuid" : "0QsUwstqQce2mSHdkxSkwA",
    "repository" : "netology_backup",
    "version_id" : 8030399,
    "version" : "8.3.3",
    "indices" : [
      "test",
      ".geoip_databases"
    ],
    "data_streams" : [ ],
    "include_global_state" : true,
    "state" : "SUCCESS",
    "start_time" : "2022-09-01T20:58:48.731Z",
    "start_time_in_millis" : 1662065928731,
    "end_time" : "2022-09-01T20:58:52.535Z",
    "end_time_in_millis" : 1662065932535,
    "duration_in_millis" : 3804,
    "failures" : [ ],
    "shards" : {
      "total" : 2,
      "failed" : 0,
      "successful" : 2
    },
    "feature_states" : [
      {
        "feature_name" : "geoip",
        "indices" : [
          ".geoip_databases"
        ]
      }
    ]
  }
}

root@server1:~/el01# docker exec -it el05 ls -l /elasticsearch-8.3.3/snapshots/
total 36
-rw-r--r-- 1 elasticsearch elasticsearch   843 Sep  1 20:58 index-0
-rw-r--r-- 1 elasticsearch elasticsearch     8 Sep  1 20:58 index.latest
drwxr-xr-x 4 elasticsearch elasticsearch  4096 Sep  1 20:58 indices
-rw-r--r-- 1 elasticsearch elasticsearch 18399 Sep  1 20:58 meta-0QsUwstqQce2mSHdkxSkwA.dat
-rw-r--r-- 1 elasticsearch elasticsearch   353 Sep  1 20:58 snap-0QsUwstqQce2mSHdkxSkwA.dat
```
- Удалите индекс `test` и создайте индекс `test-2`. **Приведите в ответе** список индексов.
```bash
root@server1:~/el01# curl -X DELETE "localhost:9200/test?pretty"
{
  "acknowledged" : true
}
root@server1:~/el01# curl -X PUT localhost:9200/test-2 -H 'Content-Type: application/json' -d'{ "settings": { "number_of_shards": 2,  "number_of_replicas": 0 }}'
{"acknowledged":true,"shards_acknowledged":true,"index":"test-2"}root@server1:~/el01#
root@server1:~/el01# curl 'localhost:9200/_cat/indices?pretty'
green open test-2 vWHGuElTTYSOQBncKYGa6A 2 0 0 0 450b 450b
```
- [Восстановите] состояние кластера `elasticsearch` из `snapshot`, созданного ранее.

```bash
root@server1:~/el01# curl -X POST "localhost:9200/_snapshot/netology_backup/snapshot_1/_restore?pretty" -H 'Content-Type: application/json' -d'
> {
>   "indices": "*",
>   "include_global_state": true
> }
> '
{
  "accepted" : true
}
root@server1:~/el01# curl 'localhost:9200/_cat/indices?pretty'
green open test   BaPNJtAJQdypSk5jDSJ4-g 1 0 0 0 225b 225b
green open test-2 vWHGuElTTYSOQBncKYGa6A 2 0 0 0 450b 450b
root@server1:~/el01#
```
