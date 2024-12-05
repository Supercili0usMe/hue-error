Опишу ошибки + способ их решения, возникшие у меня при установке и запуске контейнеров.

Сразу уточню, устанавливал на Windows 11.
## Первая ошибка
**Первая ошибка** которая у меня возникла (вернее несколько первых ошибок) выглядят следующим образом:
<p align="center">
    <img src="./img/Pasted image 20241204100013.png" width="500">
</p>

Решается данная проблема следующим образом: отключаем службу winnat (команда показана ниже) => запускаем контейнеры => включаем службу winnat (команда показана ниже)

<p align="center">
    <img src="./img/Pasted image 20241204100240.png" width="500">
</p>

## Вторая ошибка
**Вторая ошибка** выглядела следующим образом: перехожу в hue (http://localhost:8888), а там белый экран и название вкладки изменилось на "Hue - 500 - Server error". Посмотрев логи увидел следующее:
<p align="center">
    <img src="./img/Pasted image 20241204100850.png" width="500">
</p>

(на картинке написано что таблица hue_db.django_session doesn't exist)

 Решилась данная проблема у меня следующим образом:

1. Подключился к базе данных из контейнера `database`:

```bash
docker exec -it database mysql -u root -pD4EfSXVWMr84
```

2. Проверил наличие базы `hue_db` и вывел оттуда список таблиц:

```sql
SHOW DATABASES;
USE hue_db;
SHOW TABLES;

+--------------------------------+
| Tables_in_hue_db               |
+--------------------------------+
| auth_group                     |
| auth_group_permissions         |
| auth_permission                |
| auth_user                      |
| auth_user_groups               |
| auth_user_user_permissions     |
| axes_accessattempt             |
| axes_accesslog                 |
| beeswax_metainstall            |
| beeswax_queryhistory           |
| beeswax_savedquery             |
| beeswax_session                |
| desktop_document               |
| desktop_document2              |
| desktop_document2_dependencies |
| desktop_document2permission    |
| desktop_document_tags          |
| desktop_documentpermission     |
| desktop_documenttag            |
| desktop_settings               |
| desktop_userpreferences        |
| django_admin_log               |
| django_content_type            |
| django_migrations              |
| documentpermission2_groups     |
| documentpermission2_users      |
| documentpermission_groups      |
| documentpermission_users       |
+--------------------------------+
```

3. Проверил таблицу `django_migrations`, и вставил туда проблемную миграцию:

```bash
mysql> select * from django_migrations where app = "desktop";
Empty set (0.01 sec)

mysql> INSERT INTO django_migrations (app, name, applied)
    -> VALUES ('desktop', '0001_initial', NOW());
Query OK, 1 row affected (0.08 sec)

mysql> select * from django_migrations where app = "desktop";
+----+---------+--------------+----------------------------+
| id | app     | name         | applied                    |
+----+---------+--------------+----------------------------+
| 19 | desktop | 0001_initial | 2024-12-04 15:49:26.000000 |
+----+---------+--------------+----------------------------+
1 row in set (0.00 sec)
```

4. Перешел в контейнер `hue` и запустил миграцию:

```bash
PS C:\Users\supercilious> docker exec -it hue bash
root@hue:/usr/share/hue# /usr/share/hue/build/env/bin/hue migrate
```

После чего миграции были успешно выполнены:
```bash
Operations to perform:
  Apply all migrations: admin, auth, axes, beeswax, contenttypes, desktop, oozie, sessions, sites, useradmin
Running migrations:
  Applying desktop.0002_initial... OK
  Applying desktop.0003_initial... OK
  Applying desktop.0004_initial... OK
  Applying desktop.0005_initial... OK
  Applying desktop.0006_initial... OK
  Applying desktop.0007_initial... OK
  Applying desktop.0008_auto_20191031_0704... OK
  Applying desktop.0009_auto_20191202_1056... OK
  Applying desktop.0010_auto_20200115_0908... OK
  Applying desktop.0011_document2_connector... OK
  Applying desktop.0012_connector_interface... OK
  Applying oozie.0001_initial... OK
  Applying oozie.0002_initial... OK
  Applying oozie.0003_initial... OK
  Applying oozie.0004_initial... OK
  Applying oozie.0005_initial... OK
  Applying oozie.0006_auto_20200714_1204... OK
  Applying sessions.0001_initial... OK
  Applying sites.0001_initial... OK
  Applying sites.0002_alter_domain_unique... OK
  Applying useradmin.0001_initial... OK
  Applying useradmin.0002_userprofile_json_data... OK
  Applying useradmin.0003_auto_20200203_0802... OK
  Applying useradmin.0004_userprofile_hostname... OK
```

5. Перезапустил контейнер и все заработало