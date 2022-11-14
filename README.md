# OTUS ДЗ№24  SELinux - когда все запрещено. #
-----------------------------------------------------------------------
## Домашнее задание ##
### 1. Запустить nginx на нестандартном порту 3-мя разными способами: ###
* переключатели setsebool;
* добавление нестандартного порта в имеющийся тип;
* формирование и установка модуля SELinux.

#### К сдаче: ####
README с описанием каждого решения (скриншоты и демонстрация приветствуются).

### 2. Обеспечить работоспособность приложения при включенном selinux. ####
*    развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
*    выяснить причину неработоспособности механизма обновления зоны (см. README);
*    предложить решение (или решения) для данной проблемы;
*    выбрать одно из решений для реализации, предварительно обосновав выбор;
*    реализовать выбранное решение и продемонстрировать его работоспособность.
#### К сдаче: ####
*    README с анализом причины неработоспособности, возможными способами решения и обоснованием выбора одного из них;
*    исправленный стенд или демонстрация работоспособной системы скриншотами и описанием.
-----------------------------------------------------------------------
## Результат ##


### 1. Запустить nginx на нестандартном порту 3-мя разными способами. ###

#### a) переключатели setsebool ####

* Установим веб-сервер nginx командой ```yum install -y nginx```

* В файле конфигурации nginx меняем порт на 12345 - ```vi /etc/nginx/nginx.conf```. Запустить на этом порту веб-сервер не получается:
```
[root@log vagrant]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@log vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Mon 2022-11-14 05:06:19 UTC; 3s ago
  Process: 1009 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 1053 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 1052 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 1010 (code=exited, status=0/SUCCESS)

Nov 14 05:06:19 log systemd[1]: Starting The nginx HTTP and reverse proxy server...
Nov 14 05:06:19 log nginx[1053]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Nov 14 05:06:19 log nginx[1053]: nginx: [emerg] bind() to 0.0.0.0:12345 failed (13: Permission denied)
Nov 14 05:06:19 log nginx[1053]: nginx: configuration file /etc/nginx/nginx.conf test failed
Nov 14 05:06:19 log systemd[1]: nginx.service: control process exited, code=exited status=1
Nov 14 05:06:19 log systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Nov 14 05:06:19 log systemd[1]: Unit nginx.service entered failed state.
Nov 14 05:06:19 log systemd[1]: nginx.service failed.
```

* Далее выполним команду ```audit2why < /var/log/audit/audit.log```, для того чтобы понять какой логический тип включать
```
[root@log vagrant]# audit2why < /var/log/audit/audit.log
type=AVC msg=audit(1668401813.696:1119): avc:  denied  { name_bind } for  pid=3778 comm="nginx" src=12345 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
```

* Утилита ```audit2why``` предлагает выполнить команду ```setsebool -P nis_enabled 1```. После её выполнения веб-сервер nginx успешно запускается. Для отключения переключателя выполняем команду ```setsebool -P nis_enabled 0```.

#### b) добавление нестандартного порта в имеющийся тип ####

* Посмотрим, какие порты могут работать по протоколу http, для этого выполним команду ```semanage port -l | grep http```.
```
[root@log vagrant]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

* Добавляем наш нестандартный порт в правило политики, командой - ```semanage port -a -t http_port_t -p tcp 12345```. Теперь видим, что порт 12345 может работать по протоколу http:
```
[root@log vagrant]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      12345, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
* Nginx успешно запускается на нестандартном порту 1245

* Удалить порт можно командой ```semanage port -d -t http_port_t -p tcp 12345```



#### с) формирование и установка модуля SELinux ####

* Для этого нам необходимо будет скомпилировать модуль на основе лог файла аудита, в котором есть информация о запретах.

* Выполним команду ```audit2allow -M httpd_add --debug < /var/log/audit/audit.log```:
```
[root@log vagrant]# audit2allow -M httpd_add --debug < /var/log/audit/audit.log
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i httpd_add.pp
```
* Происталлируем наш созданный модуль командой ```semodule -i httpd_add.pp```

* Проверим загрузился ли наш модуль:
```
[root@log vagrant]# semodule -l | grep http
httpd_add	1.0
```

* Nginx снова работает на нестандартном порту 12345:
```
[root@log vagrant]# systemctl restart nginx
[root@log vagrant]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2022-11-14 05:25:02 UTC; 1s ago
  Process: 19900 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 19898 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 19897 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 19902 (nginx)
   CGroup: /system.slice/nginx.service
           ├─19902 nginx: master process /usr/sbin/nginx
           └─19904 nginx: worker process

Nov 14 05:25:02 log systemd[1]: Starting The nginx HTTP and reverse proxy server...
Nov 14 05:25:02 log nginx[19898]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Nov 14 05:25:02 log nginx[19898]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Nov 14 05:25:02 log systemd[1]: Started The nginx HTTP and reverse proxy server.
```

* Чтобы удалить модуль, нужно выполнить команду ```semodule -r httpd_add```. Чтобы выключить модуль ```semodule -d -v httpd_add```. Включить модуль ```semodule -e -v httpd_add```
