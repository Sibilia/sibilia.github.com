---
layout: post
title: "Установка и настройка Munin"
date: 2013-02-18 15:07
comments: true
categories: [linux]
---
[Munin](http://munin-monitoring.org/) - Это мощная клиент-серверная система мониторинга параметров серверов. Главный сервер munin запускается по cron и опрашивает munin-node сервера собирая с них данные и рисует красивые и наглядные графики. На munin-node серверах демон при подключении главного сервера запускает скрипты плагинов из /etc/munin/plugins/. Плагинов в стандартной установке большое количество. Можно написать и свои. 

### Установка 
Приведу пример установки клиента и сервера для CentOS/RHEL. Для Debian/Ubuntu всё почти аналогично.
Для начала установим главный сервер:
	# yum install munin munin-node
Для клиентов необходим только munin-node:
	# yum install munin-node
### Настройка 
Главный файл настройки сервера Munin /etc/munin/munin.conf. В нём необходимо прописать клиентов:
```
# a simple host tree
[node1]
	address 172.16.0.12
	use_node_name yes

[node2]
	address 172.16.0.13
    use_node_name yes

[bckp]
    address 127.0.0.1
    use_node_name yes
```
Для настройки клиентов нам необходимо подправить /etc/munin/munin-node.conf:
```
# Разрешаем подключаться серверу:
allow ^127\.0\.0\.1$
allow ^172\.16\.0\.10$ # ip address bckp 

# указываем host name и ip
host_name node1
host 172.16.0.12
```
Теперь включим парочку дополнительных плагинов:
	ln -s /usr/share/munin/plugins/acpi /etc/munin/plugins/
	ln -s /usr/share/munin/plugins/iostat /etc/munin/plugins/
	ln -s /usr/share/munin/plugins/iostat_ios /etc/munin/plugins/
Как видно из примера плагины подключаются созданием симлинка в /etc/munin/plugins/
##### Подключение плагинов для MongoDB
Плагины для MongoDB не идут в стандартной поставке, но их не сложно установить. 
	# wget https://github.com/erh/mongo-munin/archive/master.zip -O /tmp/mongo-munin.zip
	# unzip /tmp/mongo-munin.zip /tmp/
	# mkdir -p /usr/local/share/munin/plugins
	# cp /tmp/mongo-munin-master/mongo_* /usr/local/share/munin/plugins/
	
	# ln -s /usr/local/share/munin/plugins/mongo_btree /etc/munin/plugins/
	# ln -s /usr/local/share/munin/plugins/mongo_conn /etc/munin/plugins/
	# ln -s /usr/local/share/munin/plugins/mongo_lock /etc/munin/plugins/
	# ln -s /usr/local/share/munin/plugins/mongo_mem /etc/munin/plugins/
	# ln -s /usr/local/share/munin/plugins/mongo_ops /etc/munin/plugins/
Не забываем перезагружать munin после правки конфигов и добавления/удаления плагинов.
	# /etc/init.d/munin-node restart
Можно проверить просто запустив один из скриптов. Первоначально у меня они не отрабатывали:
	# /usr/local/share/munin/plugins/mongo_btree
	...
	  File "/usr/lib64/python2.6/json/decoder.py", line 319, in decode
	    obj, end = self.raw_decode(s, idx=_w(s, 0).end())
	  File "/usr/lib64/python2.6/json/decoder.py", line 338, in raw_decode
    	raise ValueError("No JSON object could be decoded")
	ValueError: No JSON object could be decoded
Подравив в скрипте вывод значения я выяснил в чём проблема...
	# vim /usr/local/share/munin/plugins/mongo_btree
```
28     raw = urllib2.urlopen(req).read()
29     print raw
30     return json.loads( raw )["serverStatus"]
```
	# /usr/local/share/munin/plugins/mongo_btree
	You are trying to access MongoDB on the native driver port. For http diagnostic access, add 1000 to the port number
	....
Сново правим скрипт не забыв удалить добавленную строчку на прошлом этапе.
```
    url = "http://%s:%d/_status" % (host, port+1000)
```
Теперь всё в порядке.
	# /usr/local/share/munin/plugins/mongo_btree
	missRatio.value 0
	resets.value 0
	hits.value 23325605
	misses.value 46
	accesses.value 23325651
Не забываем сделать соответствующие поправки в остальных скриптах mongo_
##### Подключение плагинов для MySQL
В стандартной поставке есть плагины для mysql, если нужно, подключаем и их:
	# ln -s /usr/share/munin/plugins/mysql_* /etc/munin/plugins
Для их работы необходимы дополнительные пакеты для perl:
	# yum install perl-Cache-Cache perl-DBD-MySQL perl-IPC-ShareLite
Если ещё чего не хватит, то это легко выяснить чтением логов /var/log/munin-node/munin-node.log.
Создаём в mysql пользователя для munin:
	# mysql -u root -p -e 'create user munin'
Подправим файл /etc/munin/plugin-conf.d/munin-node:
	[mysql*]
		user root
		env.mysqlopts --defaults-file=/etc/mysql/my.cnf
		env.mysqluser munin

##### Использование.
На главном сервере Munin необходим Web сервер. В файле /etc/munin/munin.conf прописывается куда будут генерироваться итоговые .html
Например для стандартных настроек достаточно просто в браузере перейти на ip адресс главного сервера munin: http://172.16.0.10/munin
