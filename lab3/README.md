# Отчет по лабораторной работе №3

# Содержание:

[Часть 1: Поднимаем Postgres](#часть-1-поднимаем-postgres)

[Часть 2: Проверяем репликацию](#часть-2-проверяем-репликацию)

[Часть 3: Делаем высокую доступность](#часть-3-делаем-высокую-доступность)

 ## Часть 1: Поднимаем Postgres

1. Подготавливаем Dockerfile для нашего постгреса. 
   
2. Подготавливаем compose файл, в котором описываем наш деплой постгреса. Так же добавляем в него Zookepeer.

   <details>
   <summary>Вопрос</summary>

   *Порты 8008 и 5432 вынесены в разные директивы, expose и ports. По сути, если записать 8008 в ports, то он тоже станет exposed. В чем разница?*
   
   **expose** просто объявляет, что контейнер использует данный порт внутри своей сети. Эта директива не делает порт доступным вне контейнера. **ports** публикует порт из контейнера на хосте. Эта директива используется для маппинга портов контейнера на порты хоста.
   
   </details><br>

4. Создаем `postgres0.yml` и затем на основе него — `postgres1.yml`

5. Деплоим. Проверяем в логах, что зукипер запустился, и что одна нода постгреса из двух стала лидером/овнером/мастером.
   <details>
   <summary>Изображение</summary>

   ![браузер](screenshots/1.png)
   </details><br>

   <details>
   <summary>Изображение</summary>

   ![браузер](screenshots/2.png)
   </details><br>

   <details>
   <summary>Изображение</summary>

   ![браузер](screenshots/3.png)
   </details><br>

   <details>
   <summary>Изображение</summary>

   ![браузер](screenshots/4.png)
   </details><br>

   <details>
   <summary>Вопрос</summary>

   *При обычном перезапуске композ-проекта, будет ли сбилден заново образ? А если предварительно отредактировать файлы postgresX.yml? А если содержимое самого Dockerfile? Почему?*
   
   При обычном перезапуске композ-проекта образ не будет сбилден заново, если он уже существует и не были изменен.

   При редактировании файла postgresX.yml образ не будет сбилден заново. Файл yml используется для настройки контейнера, а не для построения образа.

   Редактирование Dockerfile приведет к пересборке образа при перезапуске композ-проекта. Docker Compose обнаружит изменения в Dockerfile и запустит процесс сборки заново.
   
   </details><br>

## Часть 2: Проверяем репликацию

Настройка `pgadmin`

```yml
pgadmin:
  image: dpage/pgadmin4
  container_name: pgadmin
  environment:
    PGADMIN_DEFAULT_EMAIL: admin@admin.com
    PGADMIN_DEFAULT_PASSWORD: admin
  ports:
    - "5050:80"
```

Создание новой таблицы в `pg-master`

```sql
CREATE TABLE my_first_replication (
    id int,
    my_data varchar,
    my_comment varchar
);

INSERT INTO my_first_replication VALUES (
    '1',
    'my important data',
    'is this line replicated?'
);

```

Фиксируем успешную репликацию в `pg-slave`

![pg-slave-replication](screenshots/5.png)

Тестируем редактирование из `pg-slave`

![pg-slave-update-failed](screenshots/6.png)

 ## Часть 3: Делаем высокую доступность

1. Для балансировки трафика нам нужен специальное ПО, собственно балансировщик. Например, HAProxy — добавляем его в `docker-compose.yml`:

```yaml
  haproxy:
    image: haproxy:3.0
    container_name: postgres_entrypoint 
    ports:
      - 5432:5432 
      - 7000:7000
    depends_on: 
      - pg-master
      - pg-slave
      - zoo
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
```

2. Не забываем создать упомянутый выше `haproxy.cfg` со следующим содержимым:

   <details>
   <summary>Изображение</summary>

   ![браузер](screenshots/7.png)
   </details><br>

3. Перезапускаем проект, проверяем.

   <details>
   <summary>Изображение</summary>

   ![браузер](screenshots/8.png)
   </details><br>

4. Аналогично Пункту 1 Части 2, подключаемся к “новой” базе данных, в качестве адреса используя имя контейнера с HAProxy. Проверяем, что подключение успешное - по умолчанию оно всегда будет редиректить трафик на мастер-ноду

   <details>
   <summary>Изображение</summary>

   ![браузер](screenshots/9.png)
   </details><br>

   <details>
   <summary>Изображение</summary>

   ![браузер](screenshots/10.png)
   </details><br>

 ## Задание

1. Любым способом выключаем доступ до ноды, которая сейчас является мастером (например, через docker stop).

2. Некоторое время ждем, после этого анализируем логи. 

   <details>
   <summary>Изображение</summary>

   ![браузер](screenshots/11.png)
   </details><br>

   <details>
   <summary>Изображение</summary>

   ![браузер](screenshots/12.png)
   </details><br>

3. Так же пытаемся считать/записать что-то в БД через entrypoint подключение, все получилось, эта же строчка появляется в pg-slave.

   <details>
   <summary>Изображение</summary>

   ![браузер](screenshots/13.png)
   </details><br>

   <details>
   <summary>Изображение</summary>

   ![браузер](screenshots/14.png)
   </details><br>

4. То есть, после отключения pg-master, patroni автоматически сделал лидером pg-slave, также и HAProxy перестроился и переключил входящие запросы на нового лидера. И теперь pg-slave не read-only, а также можно записывать данные, потому что теперь это лидер.
