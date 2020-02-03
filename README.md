## Ansible-скрипты для поднятия форума.

### Описание структуры файлов

В директории `hosts` должны находиться файлы с адресами хостов (можно использовать файлы .example в качестве основы):

- `local_dev.ini` - какой-либо свой локальный сервер для тестов
- `dev.ini` - dev.omskvelo.ru
- `prod.ini` - omskvelo.ru

Основные настраиваемые переменные находятся в файле `vars/vars.yml` (скорее всего ничего менять не нужно будет).  

«Секретные» переменные находятся в директории `secret_vars`:

- `dev.yml` - для local_dev и dev хостов
- `prod.yml` -  соответственно, для prod'а

В директории `bin` находятся скрипты для удобства запуска, которые просто берут адреса хостов и секретные переменные из соответствующих файлов и запускают playbook:

- `bin/play_dev` `playbook.yml` — Запустит `playbook.yml` на dev-хосте
- `bin/play_local_dev` `playbook.yml` — на local_dev
- `bin/prod_play` `playbook.yml` — на проде

### Описания playbook'ов:

- `py2.yml` — Установить python2 на целевой машине (для ansibl'а)
- `main.yml` — Установить/настроить основное окружение (nginx/mysql/php/etc)
- `ipb.yml` — Зальёт исходники форума из гита в нужную директорию (нужен доступ к гиту)
- `db_dump.yml` — Сделать дамп базы
- `db_import.yml` `-e file=db_dump.tgz` — Залить дамп базы из файла `db_dump.tgz` на целевую машину (не стоит запускать на проде ес-но)
- `uploads_partial_dump.yml` — Частично скачать содержимое uploads (чтобы форум поставленный из дампа базы на свежую машину выглядел более-менее адекватно)
- `uploads_partial_import.ym.` `-e folder=uploads_folder` — Залить частично скачанную с помощью `uploads_partial_dump.yml` директорию uploads, из локальной директории `uploads_folder`
- `https.iml` — Выпустить https-сертификат через let's encrypt (в `secret_vars` должна быть определена переменная `yandex_pdd_token`)
- `cron.yml` `-e key=<key>` - настроить cron для IPB, key берётся из  AdminCP → System → Settings/Advanced Configuration → Server Environment → выбрать "Use cron", скопировать ключ и подставить как параметр `key`), для прода стоит сделать.


### Пример как задеплоить на local_dev:

Отступление: у [ICS](https://invisioncommunity.com/) есть ограничение на количество инсталляций, с доменным именем разрешается только две - основная и тестовая (сейчас это omskvelo.ru и dev.omskvelo.ru соответственно). А также разрешается неограниченное количество инсталляций по адресу 127.0.0.1 - что, конечно, неудобно, т.к. тогда придется устанавливать форум, БД и т.п. непосредственно на используемую ОС. Но ограничение на использование одной тестовой инсталляции можно обойти, прописав dev.omskvelo.ru с нужным адресом локально в `/etc/hosts`, именно такой вариант инсталляции описывается ниже.

1. поднимаем хост  
1. прописываем его адрес в `hosts/local_dev.ini`
1. (опционально) прописываем его адрес как dev.omskvelo.ru в `/etc/hosts` локально и на самом хосте (на всякий случай)
1. делаем/скачиваем дамп базы:  
   ```
   bin/prod_play db_dump.yml
   ````
   в текущей директории появится файл с названием навроде  `_omskvelo.ru-db_dump-20200129T170134.sql.gz`
1. делаем частичный дамп директории uploads (там есть файлы, без которых форум нормально не будет функционировать, в частности CSS):
   ```
   bin/prod_play uploads_partial_dump.yml
   ```
   в текущей директории появится директория с названием навроде 
   `_omskvelo.ru-uploads_partial-20200129T182835`
1. ```
   bin/play_local_dev main.yml
   ```
1. ```
   bin/play_local_dev ipb.yml
   ```
1. импортируем базу из файла, скачанного на шаге 3:  
   ```
   bin/play_local_dev db_import.yml -e file=db_dump.sql.gz
   ```
1. импортируем uploads из директории, скачанной на шаге 4:  
   ```
   bin/play_local_dev uploads_partial_import.yml -e folder=<_omskvelo.ru-uploads_partial-...>
   ```

