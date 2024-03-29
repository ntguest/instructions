Установка Home Assistant на одноплатные компьютеры, имеющие внутреннюю память emmc имеет много преимуществ. 
В частности, значительно возрастает скорость работы Home Assistant. 
Но такой способ имеет очень существенный недостаток - любая flash память подверженна старению и деградации, что приведет к неработоспособности как самой системы, так и одноплатника вообще. 
Как бы нас не уверяли производители, ничего вечного не существует. Самую значительную роль в деградации emmc, в случае установки Home Assistant, играет база данных. 
Причем неважно, будет это стандартная SQLite или более производительная MariaDB, любые операции с базой данных сильно изнашивают носитель. 
Существуют способы ограничить операции ввода-вывода в базу данных путем настройки рекордера, фильтрации сущностей и т.д. 
Кроме того, emmc, чаще всего, имеют не слишком большой размер, а т.к. на ней установлена и операционная система и Home Assistant, то для самой базы может остаться совсем мало места.

Я использую внешнюю базу данных, которая установлена на сервере Oracle Cloud. 
Плюсы этого решения заключаются в том, что на работу самой базы данных не расходуются достаточно ограниченные вычислительные ресурсы одноплатника и "вечный" носитель, о работоспособности которого заботится компания Oracle. 
Минусы тоже есть - во первых при нестабильном интернет соединении некоторые данные не смогут быть обработаны и не попадут в саму базу. 
Также многие пользователи считают, что собственный сервер, который не зависит от интернет соединения или воли поставщика вычислительных мощностей должен быть в приоритете. 
Каждый выбирает то, что считает приемлемым для своего применения.

Есть другой вариант - то перемещение базы данных, установленной на одноплатнике на карту памяти, которая имеется на всех одноплатниках. 
Также это может быть как USB флешка, так и внешний SSD диск. Мне больше по душе карта памяти, которую мы отправим "умирать" во благо нашего умного дома. Она имеет малую стоимость и практически всегда есть в наличии на замену. 
Если делать бекапы, то данные, которые "умрут" вместе с картой будут зависить только от частоты бекапов.
Решение, предлагаемое здесь, позволяет избавиться сразу от нескольких недостатков.  Во-первых, это большой размер для базы. Во-вторых продление срока службы emmc и самого одноплатника.

Если вам подходит такое решение, можно начинать.

# Установка базы банных
- Переключаемся на суперпользователя.
В зависимости от системы это либо 
```bash
su -
```
либо 
```bash
sudo su -
```
- Прежде всего, обновляем систему
```bash
apt update && apt upgrade -y && apt autoremove -y
```

- Устанавливаем MariaDB
```bash
apt install mariadb-server -y
```
И требуемые пакеты
```bash
apt-get install libmariadb-dev libmariadb-dev-compat -y
```
- Сразу останавливаем базу
```bash
systemctl stop mysql || systemctl stop mariadb
```
# Перенос базы данных
- Создаем папку, в которую будет монтироваться внешний носитель
```bash
md /dbase
```
- Смотрим название раздела
```bash
lsblk
fdisk -l
```
*сделаем вид, что карта памяти называется sdb1*
- Смотрим UID пользователя mysql
```bash
id -u mysql
```
*сделаем вид, что команда вернула значение 106*

- Отредактируем файл fstab. Для этого запустим редактов nano
```bash
nano etc/fstab
```
В открывшемся редакторе добавим в конце строку
```bash
/dev/sdb1 /dbase vfat uid=106
```
*Для записи изменеий и выхода нажимаем CTRL+O, Enter и CTRL+Z*

- Копируем все файлы данных MariaDB в созданную папку
```bash
rsync -av /var/lib/mysql /dbase
```
- **Переименуем старую папку /var/lib/mysql, сохраним её на случай сбоя**
```bash
mv /var/lib/mysql /var/lib/mysql.bak
```
# Настройка MariaDB
- Редактируем файл конфигурации MariaDB
```bash
nano /etc/mysql/mariadb.conf.d/50-server.cnf
```
В секциии [mysqld] находим и изменяем параметры datadir и bind-address
```bash
[mysqld]
datadir         = /dbase/mysql
bind-address    = 0.0.0.0
```
- Настроим AppArmor, чтобы предоставить MySQL право на изменение нового каталога.
```bash
nano /etc/apparmor.d/tunables/alias
```
Добавим правило
```bash
alias /var/lib/mysql/ -> /dbase/mysql/,
```
Перезапусткаем AppArmor
```bash
systemctl restart apparmor
```
- Запускаем MariaDB
```bash
systemctl start mysql || systemctl start mariadb
```

- Заходим в консоль MariaDB
```bash
mysql -u root -p
```
*вводим пароль администратора*

Добавляем в базу таблицу, пользователя и устанавливаем права
```bash
CREATE DATABASE <database_name> CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci; 
CREATE USER '<database_user_name>'@'localhost' IDENTIFIED BY '<database_user_password>'; 
grant all privileges on *.* to <database_user_name>@"%" identified by "<database_user_password>"; 
quit
```
*Enter после каждой строчки, вместо <database_name>, <database_user_name>, <database_user_password> вводим ваше имя базы, имя пользователя и пароль*

- Перезагрузка
```bash
reboot
```

# Настройка Home Assistant

- Изменяем в configuration.yaml 

```bash
recorder:
  db_url: mysql://<database_user_name>:<database_user_password>@<mysql_server_ip>/<database_name>?charset=utf8mb4
```

- Добавляем датчик размера базы через штатную интеграцию SQL и настроив ее следующим образом:

URL-адрес базы данных
```bash
mysql://<database_user_name>:<database_user_password>@<mysql_server_ip>/<database_name>?charset=utf8mb4
```
Запрос
```bash
SELECT table_schema "database", Round(Sum(data_length + index_length) / 1024 / 1024, 1) "value" FROM information_schema.tables WHERE table_schema="<database_name>" GROUP BY table_schema;
```
Столбец
```bash
value
```

Примерно как здесь:

![image](https://user-images.githubusercontent.com/69485846/177888525-249279d1-e186-477b-a470-6ddfe77f3409.png)





[![Donate](https://img.shields.io/badge/donate-Beer-yellow.svg)](https://www.buymeacoffee.com/ntguest)
[![Donate](https://img.shields.io/badge/donate-Yandex-blueviolet.svg)](https://yoomoney.ru/to/410011383527168)
[![Donate](https://img.shields.io/badge/ask_in-Telegram-blue.svg)](https://t.me/avkulikoff)
