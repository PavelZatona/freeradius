# freeradius
ubuntu 16.04 + fereeradius +mysql

# REQUIREMENTS:
Дистрибутив ubuntu server 16.04.4 TLS
FreeRADIUS Version 2.2.8, for host x86_64-pc-linux-gnu, built on Jul 26 2017
mysql  Ver 14.14 Distrib 5.7.22, for Linux (x86_64)

# INSTALATION

1.Установка VirtualBox v5.2
-2Gb оперативки
-25Gb физически выделенного места и создание VHD
-тип сетевого подключения bridge

2.Параметры установки ОС
-минимальная установка + сервер ssh
-ручная разметка раздела
     
	/boot 1Gb xfs отдельный раздел
    
	LVM-radius – все свободное место
	    LG1  
	      LV_root  /     5Gb  ex4 
	      LV_swap  /swap 2Gb  ex4
	      LV_var   /var  2Gb  ex4
	      LV_home  /home 2Gb  ex4

3.Обновление пакетов системы после запуска:

sudo apt-get update
sudo apt-get upgrade


4.Установка необходимого ПО:

sudo install mysql
sudo apt-get install freeradius freeradius-mysql
sudo apt-get update

5.Подготовка mysql

mysql -u root -p
create database radius;
grant all privileges  on radius.* to radius@localhost identified by "radius”;
grant all privileges  on  * . * to 'radius'@'localhost';
flush privileges;
exit

5.1 Переходим в директорию с шаблонами и экспотрируем их в БД radius:

cd etc/freeradius/sql/mysql/

mysql -u root -p radius yourdatabase < /etc/freeradius/sql/mysql/schema.sql
mysql -u root -p radius yourdatabase < /etc/freeradius/sql/mysql/nas.sql


5.2 Добавляем доп-е атрибуты для пользователей

 mysql -u root -p
 
 use radius;
 INSERT INTO radcheck (UserName, Attribute, Value) VALUES ('ivan', 'User-Password', 'password');
 exit


#INSERT INTO radcheck (username, attribute, op, value) VALUES ('thisuser', 'User-Password', ':=', 'thispassword');


6. Настройка конфигов freeradius.Открываем файл настроек Freeradius для MySQL /etc/freeradius/sql.conf и редактируем строки до такого вида:
sql { 
  database = "mysql" 
	driver = "rlm_sql_${database}"
	server = "localhost"
	login = "radius"
	password = "radius"
	radius_db = "radius" 
	#uncomment read_groups 
	read_groups = yes 
	#uncomment readclients 
	readclients = yes 
}


6.1 Далее открываем файл сайта Freeradius  /etc/freeradius/sites-enabled/default и в следующих полях раскомментируем строки sql.Приводим следующие строки к виду:

Uncomment sql on authorize{}
# See “Authorization Queries” in sql.conf
sql
...
Uncomment sql on accounting{}
# See “Accounting queries” in sql.conf
sql
...
Uncomment sql on session{}
# See “Simultaneous Use Checking Queries” in sql.conf
sql
...
Uncomment sql on post-auth{}
# See “Authentication Logging Queries” in sql.conf
sql
…

6.2 В файле /etc/freeradius/clients.conf в секции “client localhost“ нужно сделать следующее:

ipaddr = 127.0.0.1 
secret = secret 
nastype = other

6.3 Далее правим основной конфигурационный файл Freeradius и включаем поддержку Mysql  /etc/freeradius/radiusd.conf раскомментировав строку:

$INCLUDE sql.conf

7.Тестирование работы сервера. Откроем 2 ssh окна терминала, в первом остановим сервис Freeradius:

freeradius stop
sudo service freeradius stop

Затем запускаем freeradius  в режиме отладки:
sudo freeradius -X - debug mode

7.2 Во втором окне терминала отправляем тестовый запрос:

radtest ivan passwors localhost 1812 secret

##Для вывода дополнительных параметров пользователя нужно добавить AV пары в таблицу radcheck и повторим тестовый запрос

8.Тестирование функции accaunting 

8.1 Создаем и отправляем тестовый пакет для отправки на сервер, в нем размещается набор AV пар
  
radclient 127.0.0.1 auto secret -f acct_start_01.txt -x

8.2 Проверяем, что в базе mysql    появились данные запроса в таблице radacct.

8.3 Создаем еще три служебных пакета для для тестирования контроля сессий пользователей start.txt, interim-update.txt, stop.txt.

radclient 127.0.0.1 auto secret -f start.txt -x 
radclient 127.0.0.1 auto secret -f interim-update.txt -x 
radclient 127.0.0.1 auto secret -f stop.txt -x 

# 9. Для использования счетчиков(нет в задании): 

# 9.1 В файле /etc/raddb/sql/mysql/counter.conf в конце уже определен счетчик «noresetcounter», мы его подредактируем:

sqlcounter noresetcounter {
counter-name = Session-Timeout
check-name = Session-Timeout
reply-name = Session-Timeout
sqlmod-inst = sql
key = User-Name
reset = never
query = "SELECT IFNULL(SUM(AcctSessionTime),0) FROM radacct WHERE UserName='%{${key}}'"
}

# 9.2 В файле /etc/raddb/sql/mysql/dialup.conf для работы с Simultaneous-Use раскомментируем следующее:

simul_count_query = "SELECT COUNT(*) \
   FROM ${acct_table1} \
   WHERE username = '%{SQL-User-Name}' \
   AND acctstoptime IS NULL"


# Конфигурационные файлы freeradius:

/etc/freeradius/sql.conf
/etc/freeradius/sites-enabled/default
/etc/freeradius/radiusd.conf
/etc/freeradius/sql/mysql/counter.conf 
/etc/freeradius/clients.conf
/etc/freeradius/sql/mysql/schema.sql
/etc/freeradius/sql/mysql/nas.sql
/etc/freeradius/sql/mysql/admin.sql
/etc/freeradius/sql/mysql/dialup.conf

User authenticated successfully. Далее запускается секция section, в ней [sql] определяется, активна ли сессия в данный момент, если да, значит кто-то уже залогинился и в доступе будет отказано. Сам запрос описан в файле /etc/freeradius/sql/mysql/dialup.conf.



Секция post-auth логирует успешную аутентификацию в базу данных, в таблицу radpostauth. После этого отправляется ответ.

Полезные ссылки:
# http://www.pvsm.ru/mysql/109012
# https://www.opennet.ru/base/cisco/radius.txt.html
# https://wiki.hydra-billing.ru/pages/viewpage.action?pageId=26214716#id-%D0%9A%D0%BB%D0%B0%D1%81%D1%81%D0%B8%D1%87%D0%B5%D1%81%D0%BA%D0%B0%D1%8F%D0%B0%D0%B2%D1%82%D0%BE%D1%80%D0%B8%D0%B7%D0%B0%D1%86%D0%B8%D1%8F%D0%BD%D0%B0%D0%BF%D1%80%D0%B8%D0%BC%D0%B5%D1%80%D0%B5MPD-%D0%A2%D0%B5%D1%81%D1%82%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5%D0%B0%D0%BA%D0%BA%D0%B0%D1%83%D0%BD%D1%82%D0%B8%D0%BD%D0%B3%D0%B0%D1%81%D0%BF%D0%BE%D0%BC%D0%BE%D1%89%D1%8C%D1%8Eradclient
# https://github.com/FreeRADIUS/freeradius-server/blob/v3.0.x/raddb/clients.conf
# https://habr.com/post/275155/
# http://ittraveler.org/ustanovka-i-nastrojka-radius-servera-na-ubuntu-s-veb-interfejsom/
# https://www.vpsserver.com/community/tutorials/10/setup-and-configuration-of-freeradius-mysql-on-ubuntu-14-04-64bit/
