# Отчет по лабораторной работе №3

# Содержание:

[Часть 1: Поднимаем Postgres](#часть-1-поднимаем-postgres)

 ## Часть 1: Поднимаем Postgres

1. Подготавливаем Dockerfile для нашего постгреса. 
   
2. Подготавливаем compose файл, в котором описываем наш деплой постгреса. Так же добавляем в него Zookepeer.

   <details>
   <summary>Вопрос</summary>

   *Порты 8008 и 5432 вынесены в разные директивы, expose и ports. По сути, если записать 8008 в ports, то он тоже станет exposed. В чем разница?*
   
   **expose** просто объявляет, что контейнер использует данный порт внутри своей сети. Эта директива не делает порт доступным вне контейнера. **ports** публикует порт из контейнера на хосте. Эта директива используется для маппинга портов контейнера на порты хоста.
   
   </details><br>

3. Создаем `postgres0.yml` и затем на основе него — `postgres1.yml`

4. Деплоим. Проверяем в логах, что зукипер запустился, и что одна нода постгреса из двух стала лидером/овнером/мастером.
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