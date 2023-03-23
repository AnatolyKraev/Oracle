# Создание тестовой базы Oracle в инфраструктуре Яндекс Облака

Так как готового решения от Яндекса для создания базы данных Oracle нет, её можно создать самостоятельно на виртуальной машине. В основном это нужно для тестирования сценариев Data Transfer.

## Установка и настройка

1. [Создайте виртуальную машину](https://cloud.yandex.ru/docs/compute/operations/vm-create/create-linux-vm) в Яндекс Облаке с операционной системой [CentOS 7](https://cloud.yandex.ru/marketplace/products/yc/centos-7) со следующими характеристиками:

    * **vCPU** — 4.
    * **RAM** — 4 ГБ.
    * **Тип диска** — NR SSD (на других типах дисков БД не поднимается).
    * **Публичный адрес** — Автоматически.

1. [Подключитесь к ней по SSH](https://cloud.yandex.ru/docs/compute/operations/vm-connect/ssh#vm-connect).

1. Перейдите в режим выполнения команд от имени `root`:

    ```bash
    sudo -s
    ```

1. Скачайте и установите дистрибутив Oracle Database Preinstallation версии 7:

    ```bash
    curl -o oracle-database-preinstall-21c-1.0-1.el7.x86_64.rpm \
      https://yum.oracle.com/repo/OracleLinux/OL7/latest/x86_64/getPackage/oracle-database-preinstall-21c-1.0-1.el7.x86_64.rpm && \
    yum -y localinstall oracle-database-preinstall-21c-1.0-1.el7.x86_64.rpm
    ```

1. Скачайте и установите дистрибутив Oracle Database XE версии 7:

    ```bash
    curl -o oracle-database-xe-21c-1.0-1.ol7.x86_64.rpm \
      https://download.oracle.com/otn-pub/otn_software/db-express/oracle-database-xe-21c-1.0-1.ol7.x86_64.rpm && \
    yum -y localinstall oracle-database-xe-21c-1.0-1.ol7.x86_64.rpm
    ```

    Если загрузка файла с сайта Oracle будет запрещена, скачайте файл [oracle-database-xe-21c-1.0-1.ol7.x86_64.rpm](https://download.oracle.com/otn-pub/otn_software/db-express/oracle-database-xe-21c-1.0-1.ol7.x86_64.rpm) через VPN и [загрузите](https://cloud.yandex.ru/docs/storage/operations/objects/upload) в бакет [Object Storage](https://cloud.yandex.ru/docs/storage/) с [публичным доступом](https://cloud.yandex.ru/docs/storage/operations/buckets/bucket-availability), чтобы затем скачать на виртуальную машину:

    ```bash
    curl -o oracle-database-xe-21c-1.0-1.ol7.x86_64.rpm \
      https://storage.yandexcloud.net/<имя бакета в Object Storage>/oracle-database-xe-21c-1.0-1.ol7.x86_64.rpm && \
    yum -y localinstall oracle-database-xe-21c-1.0-1.ol7.x86_64.rpm
    ```

1. Задайте пароль для администратора БД, это пароль для административных аккаунтов `SYS`, `SYSTEM` и `PDBADMIN`:

    ```bash
    /etc/init.d/oracle-xe-21c configure
    ```

1. Настройте переменные окружения Oracle Database XE:

    ```bash
    export ORACLE_SID=XE && \
    export ORAENV_ASK=NO && \
    . /opt/oracle/product/21c/dbhomeXE/bin/oraenv
    ```

    На вопрос `ORACLE_HOME = [] ?` введите `/opt/oracle/product/21c/dbhomeXE`

    Чтобы этот шаг не приходилось делать каждый раз при повторном подключении к виртуальной машине, внесите переменные окружения в файл `.bash_profile`:

    1. Установите текстовый редактор nano:

        ```bash
        yum install nano
        ```

    1. Внесите переменные в файл `.bash_profile`:

        ```bash
        nano .bash_profile
        ```

        Пример содержимого файла:

        ```text
        # .bash_profile

        # Get the aliases and functions
        if [ -f ~/.bashrc ]; then
                . ~/.bashrc
        fi

        # User specific environment and startup programs

        PATH=$PATH:$HOME/.local/bin:$HOME/bin

        export PATH
        export ORACLE_SID=XE
        export ORAENV_ASK=NO
        . /opt/oracle/product/21c/dbhomeXE/bin/oraenv
        ```

1. Чтобы проверить статус лисенера ([listener](https://docs.oracle.com/cd/E11882_01/network.112/e41945/listenercfg.htm#NETAG010)) и базы данных, выполните команду:

    ```bash
    /etc/init.d/oracle-xe-21c status
    ```

    Должен быть результат:

    ```bash
    Status of the Oracle XE 21c service:

    LISTENER status: RUNNING
    XE Database status:   RUNNING
    ```

1. Войдите в базу данных, используя заданный пароль для администратора:

    ```bash
    sqlplus SYS AS SYSDBA
    ```

## Переключении в PDB

По умолчанию при подключении к базе данных Oracle выполняется вход в корневой контейнер (CDB), а не в подключаемую базу данных (PDB). Подробнее о том, что такое CDB и PDB читайте в [документации Oracle](https://docs.oracle.com/en/database/oracle/oracle-database/21/cncpt/CDBs-and-PDBs.html). При этом работать и экспортировать данные необходимо именно из PDB. Для этого:

1. Проверьте, куда был выполнен вход:

    ```sql
    SHOW CON_NAME
    ```

    Если результат `CDB$ROOT`, необходимо подключиться к PDB.

1. Проверьте список доступных баз для подключения:

    ```sql
    SHOW PDBS
    ```

    Результатом должно быть две базы: `PDB$SEED` и `XEPDB1`. Нас интересует вторая.

1. Смените сессию подключения на базу `XEPDB1`:

    ```sql
    ALTER SESSION SET container = XEPDB1;
    ```

1. Убедитесь, что теперь вход выполнен в `XEPDB1`:

    ```sql
    SHOW CON_NAME
    ```

    Если результат `XEPDB1`, можно приступать к созданию пользователя.

## Создание пользователя

Для выполнения трансфера администратор SYS не подходит, согласно инструкции по [подготовке к трансферу для Oracle](https://cloud.yandex.ru/docs/data-transfer/operations/prepare#source-oracle) нужен обычный пользователь. Чтобы создать такого пользователя:

1. Подключитесь от имени администратора к подключаемой базе данных `XEPDB1`:

    ```bash
    sqlplus SYS/<пароль администратора>@localhost/XEPDB1 AS SYSDBA
    ```

1. Создайте пользователя и выдайте ему права на подключение к базе:

    ```bash
    CREATE USER <имя пользователя> IDENTIFIED BY <пароль>;
    GRANT CREATE SESSION TO <имя пользователя>;
    ```

1. Выдайте пользователю права на создание таблиц:

    ```sql
    GRANT CREATE TABLE TO <имя пользователя>;
    ```

1. Снимите ограничение с квоты пользователя в табличном пространстве `USERS`:

    ```sql
    ALTER USER <имя пользователя> quota unlimited on USERS;
    ```

1. Проверьте возможность подключения от имени пользователя:

    ```bash
    sqlplus <имя пользователя>/<пароль>@localhost/XEPDB1
    ```

## Удалённое подключение

Для удаленного подключения установите на локальной машине `sqlplus` (пример для Ubuntu):

1. Скачайте дистрибутивы с [сайта](https://www.oracle.com/database/technologies/instant-client/linux-x86-64-downloads.html):

    ```bash
    wget https://download.oracle.com/otn_software/linux/instantclient/218000/oracle-instantclient-basic-21.8.0.0.0-1.x86_64.rpm \
         https://download.oracle.com/otn_software/linux/instantclient/218000/oracle-instantclient-sqlplus-21.8.0.0.0-1.x86_64.rpm \
         https://download.oracle.com/otn_software/linux/instantclient/218000/oracle-instantclient-devel-21.8.0.0.0-1.x86_64.rpm
    ```

1. Установите `alien`:

    ```bash
    sudo apt-get install alien
    ```

1. Установите с помощью `alien` скачанные дистрибутивы:

    ```bash
    sudo alien -i oracle-instantclient-basic-21.8.0.0.0-1.x86_64.rpm && \
    sudo alien -i oracle-instantclient-sqlplus-21.8.0.0.0-1.x86_64.rpm && \
    sudo alien -i oracle-instantclient-devel-21.8.0.0.0-1.x86_64.rpm
    ```

1. Установите `libaio.so`:

    ```bash
    sudo apt-get install libaio1
    ```

1. Создайте конфигурационный файл Oracle:

    ```bash
    sudo vim /etc/ld.so.conf.d/oracle.conf
    ```

1. Вставьте в него строчку `/usr/lib/oracle/21/client/lib/`.

1. Обновите конфигурацию:

    ```bash
    sudo ldconfig
    ```

1. Проверьте возможность подключения:

    ```bash
    sqlplus SYS/<пароль администратора>@<публичный IP-адрес виртуальной машины>:1521/XEPDB1 AS SYSDBA
    ```

    Или

    ```bash
    sqlplus <имя пользователя>/<пароль>@<публичный IP-адрес виртуальной машины>:1521/XEPDB1
    ```

## Наполнение базы данных тестовыми данными

Для избежания проблем с правами к созданным данным, создавайте их от имени пользователя, с помощью которого трансфер будет подключаться к источнику.

1. Подключитесь к базе данных от имени пользователя локально или удаленно:

    ```bash
    sqlplus <имя пользователя>/<пароль>@localhost/XEPDB1
    ```

    Или:

    ```bash
    sqlplus <имя пользователя>/<пароль>@<публичный IP-адрес виртуальной машины>:1521/XEPDB1
    ```

1. Создайте таблицу `emp` с первичным ключом в столбце `empno`:

    ```sql
    CREATE TABLE emp (
        empno             NUMBER(5) PRIMARY KEY,
        ename             VARCHAR2(15),
        ssn               NUMBER(9),
        job               VARCHAR2(10)
    );
    ```

1. Вставьте три строки:

    ```sql
    INSERT ALL
        INTO emp VALUES (1, 'Juan', 11, 'painting')
        INTO emp VALUES (2, 'Ivan', 22, 'nothing')
        INTO emp VALUES (3, 'Anatoly', 33, 'writing')
    SELECT * FROM dual;
    ```

1. Зафиксируйте изменения:

    ```sql
    COMMIT;
    ```

Теперь вы можете использовать созданную БД для выполнения [трансфера с источником Oracle](https://cloud.yandex.ru/docs/data-transfer/operations/endpoint/source/oracle).
