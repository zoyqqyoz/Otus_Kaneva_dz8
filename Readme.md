DZ8. Инициализация системы, Systemd

Цели занятия

понимать различие систем инициализации;
использовать основные утилиты systemd;
изучить состав и синтаксис systemd unit;

1. Написать сервис, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова. Файл и слово должны задаваться в /etc/sysconfig
Создаём файл с конфигурацией для сервиса в директории /etc/sysconfig - из неё сервис будет брать необходимые переменные
```
nano /etc/sysconfig/watchlog

# Configuration file for my watchlog service
# Place it to /etc/sysconfig

# File and word in that file that we will be monit
WORD="ALERT"
LOG=/var/log/watchlog.log
```

Затем создаем /var/log/watchlog.log и пишем туда строки на своё усмотрение,
плюс ключевое слово ‘ALERT’
```
nano /var/log/watchlog.log
```

Создадим скрипт
```
nano /opt/watchlog.sh

#!/bin/bash

WORD=$1
LOG=$2
DATE=`date`

if grep $WORD $LOG &> /dev/null
then
logger "$DATE: I found word, Master!"
else
exit 0
fi
```

Делаем скрипт исполняемым:
```
chmod +x /opt/watchlog.sh
```

Создаем unit для сервиса:
```
nano /etc/systemd/system/watchlog.service

[Unit]
Description=My watchlog service

[Service]
Type=oneshot
EnvironmentFile=/etc/sysconfig/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG
```

Создаем unit для таймера
```
nano /etc/systemd/system/watchlog.timer

[Unit]
Description=Run watchlog script every 30 second

[Timer]
# Run every 30 second
OnUnitActiveSec=30
Unit=watchlog.service

[Install]
WantedBy=multi-user.target
```

Стартуем таймер и проверяем логи 
```
systemctl start watchlog.timer

tail -f /var/log/messages

Jun  1 13:54:28 otusrpm systemd[1]: Started Run watchlog script every 30 second.
Jun  1 13:54:40 otusrpm systemd[1]: Starting My watchlog service...
Jun  1 13:54:40 otusrpm root[2362]: Thu Jun  1 13:54:40 UTC 2023: I found word, Master!
Jun  1 13:54:40 otusrpm systemd[1]: watchlog.service: Succeeded.
Jun  1 13:54:40 otusrpm systemd[1]: Started My watchlog service.
Jun  1 13:55:40 otusrpm systemd[1]: Starting My watchlog service...
Jun  1 13:55:40 otusrpm root[2370]: Thu Jun  1 13:55:40 UTC 2023: I found word, Master!
Jun  1 13:55:40 otusrpm systemd[1]: watchlog.service: Succeeded.
```

2. Из epel установить spawn-fcgi и переписать init-скрипт на unit-файл. Имя сервиса должно также называться.

Устанавливаем spawn-fcgi и необходимые для него пакеты:
```
yum install epel-release -y && yum install spawn-fcgi php php-cli mod_fcgid httpd -y

Installed:
  apr-1.6.3-12.el8.x86_64                           apr-util-1.6.1-6.el8.x86_64                           apr-util-bdb-1.6.1-6.el8.x86_64                                  apr-util-openssl-1.6.1-6.el8.x86_64
  centos-logos-httpd-85.8-2.el8.noarch              httpd-2.4.37-43.module_el8.5.0+1022+b541f3b1.x86_64   httpd-filesystem-2.4.37-43.module_el8.5.0+1022+b541f3b1.noarch   httpd-tools-2.4.37-43.module_el8.5.0+1022+b541f3b1.x86_64
  mailcap-2.1.48-3.el8.noarch                       mod_fcgid-2.3.9-17.el8.x86_64                         mod_http2-1.15.7-3.module_el8.4.0+778+c970deab.x86_64            nginx-filesystem-1:1.14.1-9.module_el8.0.0+184+e34fea82.noarch
  php-7.2.24-1.module_el8.2.0+313+b04d0a66.x86_64   php-cli-7.2.24-1.module_el8.2.0+313+b04d0a66.x86_64   php-common-7.2.24-1.module_el8.2.0+313+b04d0a66.x86_64           php-fpm-7.2.24-1.module_el8.2.0+313+b04d0a66.x86_64
  spawn-fcgi-1.6.3-17.el8.x86_64

Complete!
```

Редактируем /etc/sysconfig/spawn-fcgi: нам нужно раскомментировать строки с переменными
```
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -P /var/run/spawn-fcgi.pid -- /usr/bin/php-cgi"
```

Создаём юнит для spawn-fcgi.service
```
nano /etc/systemd/system/spawn-fcgi.service

[Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/sysconfig/spawn-fcgi
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target
```

Стартуем и проверяем стаутс сервиса spawn-fcgi.service
```
[root@otusrpm ~]# systemctl start spawn-fcgi
[root@otusrpm ~]# systemctl status spawn-fcgi
● spawn-fcgi.service - Spawn-fcgi startup service by Otus
   Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2023-06-01 14:37:47 UTC; 7s ago
 Main PID: 3352 (php-cgi)
    Tasks: 33 (limit: 12421)
   Memory: 18.5M
   CGroup: /system.slice/spawn-fcgi.service
           ├─3352 /usr/bin/php-cgi
           ├─3353 /usr/bin/php-cgi
           ├─3354 /usr/bin/php-cgi
           ├─3355 /usr/bin/php-cgi
           ├─3356 /usr/bin/php-cgi
           ├─3357 /usr/bin/php-cgi
           ├─3358 /usr/bin/php-cgi
           ├─3359 /usr/bin/php-cgi
           ├─3360 /usr/bin/php-cgi
           ├─3361 /usr/bin/php-cgi
           ├─3362 /usr/bin/php-cgi
           ├─3363 /usr/bin/php-cgi
           ├─3364 /usr/bin/php-cgi
           ├─3365 /usr/bin/php-cgi
           ├─3366 /usr/bin/php-cgi
           ├─3367 /usr/bin/php-cgi
           ├─3368 /usr/bin/php-cgi
           ├─3369 /usr/bin/php-cgi
           ├─3370 /usr/bin/php-cgi
           ├─3371 /usr/bin/php-cgi
           ├─3372 /usr/bin/php-cgi
           ├─3373 /usr/bin/php-cgi
           ├─3374 /usr/bin/php-cgi
           ├─3375 /usr/bin/php-cgi
           ├─3376 /usr/bin/php-cgi
           ├─3377 /usr/bin/php-cgi
           ├─3378 /usr/bin/php-cgi
           ├─3379 /usr/bin/php-cgi
           ├─3380 /usr/bin/php-cgi
           ├─3381 /usr/bin/php-cgi
           ├─3382 /usr/bin/php-cgi
           ├─3383 /usr/bin/php-cgi
           └─3384 /usr/bin/php-cgi

Jun 01 14:37:47 otusrpm systemd[1]: Started Spawn-fcgi startup service by Otus.
```

3. Дополнить юнит-файл apache httpd возможностью запустить несколько инстансов сервера с разными конфигами. 
Для запуска нескольких экземпляров сервиса будем использовать шаблон в
конфигурации файла окружения (/usr/lib/systemd/system/httpd.service ). Копируем шаблон в /etc/systemd/system/ и переименовываем в /etc/systemd/system/httpd@.service
```
[root@otusrpm ~]# cp /usr/lib/systemd/system/httpd.service /etc/systemd/system
[root@otusrpm ~]# mv /etc/systemd/system/httpd.service  /etc/systemd/system/httpd@.service
```

Редактируем шаблон, добавив в него строку EnvironmentFile=/etc/sysconfig/httpd-%I - указание на файлы окружения
```


[Unit]
Description=The Apache HTTP Server
Wants=httpd-init.service

After=network.target remote-fs.target nss-lookup.target httpd-init.service

Documentation=man:httpd.service(8)

[Service]
Type=notify
Environment=LANG=C
EnvironmentFile=/etc/sysconfig/httpd-%I
ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND
ExecReload=/usr/sbin/httpd $OPTIONS -k graceful
# Send SIGWINCH for graceful stop
KillSignal=SIGWINCH
KillMode=mixed
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Создаём файлы окружения:
```
nano /etc/sysconfig/httpd-first

# /etc/sysconfig/httpd-first
OPTIONS=-f conf/first.conf

nano /etc/sysconfig/httpd-second

# /etc/sysconfig/httpd-second
OPTIONS=-f conf/second.conf
```

Создаём конфиги, копируя целиком основной конфиг httpd, добавляю в каждый указание на свой pid, во втором изменяем порт на 8080
```
nano /etc/httpd/conf/first.conf

PidFile /var/run/httpd-first.pid
Listen 80

nano /etc/httpd/conf/second.conf

PidFile /var/run/httpd-second.pid
Listen 8080
```
Запускаем сервисы, проверяем статус:
```
[root@otusrpm conf]# systemctl daemon-reload
[root@otusrpm conf]# systemctl start httpd@first
[root@otusrpm conf]# systemctl start httpd@second
[root@otusrpm conf]# systemctl status httpd@first.service
● httpd@first.service - The Apache HTTP Server
   Loaded: loaded (/etc/systemd/system/httpd@.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2023-06-02 07:57:55 UTC; 57s ago
     Docs: man:httpd.service(8)
 Main PID: 4132 (httpd)
   Status: "Running, listening on: port 80"
    Tasks: 214 (limit: 12421)
   Memory: 27.7M
   CGroup: /system.slice/system-httpd.slice/httpd@first.service
           ├─4132 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─4133 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─4134 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─4135 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─4136 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           └─4137 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND

Jun 02 07:57:55 otusrpm systemd[1]: Starting The Apache HTTP Server...
Jun 02 07:57:55 otusrpm httpd[4132]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1. Set the 'ServerName' directive globally to suppress this message
Jun 02 07:57:55 otusrpm systemd[1]: Started The Apache HTTP Server.
Jun 02 07:57:55 otusrpm httpd[4132]: Server configured, listening on: port 80
[root@otusrpm conf]# systemctl status httpd@second.service
● httpd@second.service - The Apache HTTP Server
   Loaded: loaded (/etc/systemd/system/httpd@.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2023-06-02 07:58:13 UTC; 51s ago
     Docs: man:httpd.service(8)
 Main PID: 4366 (httpd)
   Status: "Running, listening on: port 8080"
    Tasks: 214 (limit: 12421)
   Memory: 37.1M
   CGroup: /system.slice/system-httpd.slice/httpd@second.service
           ├─4366 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─4367 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─4368 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─4369 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─4370 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           └─4371 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND

Jun 02 07:58:13 otusrpm systemd[1]: Starting The Apache HTTP Server...
Jun 02 07:58:13 otusrpm httpd[4366]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1. Set the 'ServerName' directive globally to suppress this message
Jun 02 07:58:13 otusrpm systemd[1]: Started The Apache HTTP Server.
Jun 02 07:58:13 otusrpm httpd[4366]: Server configured, listening on: port 8080


ss -tnulp | grep httpd

[root@otusrpm conf]# ss -tnulp | grep httpd
tcp     LISTEN   0        128                    *:8080                *:*       users:(("httpd",pid=4371,fd=4),("httpd",pid=4370,fd=4),("httpd",pid=4369,fd=4),("httpd",pid=4368,fd=4),("httpd",pid=4366,fd=4))
tcp     LISTEN   0        128                    *:80                  *:*       users:(("httpd",pid=4137,fd=4),("httpd",pid=4136,fd=4),("httpd",pid=4135,fd=4),("httpd",pid=4134,fd=4),("httpd",pid=4132,fd=4))
```



