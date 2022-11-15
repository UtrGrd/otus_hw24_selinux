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


### 2. Обеспечить работоспособность приложения при включенном selinux. ####

Проанализируем логи с помощью утилит audit2why.
```
[root@ns01 vagrant]# audit2why < /var/log/audit/audit.log
type=AVC msg=audit(1668519591.094:1992): avc:  denied  { create } for  pid=5315 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1668519621.681:1993): avc:  denied  { create } for  pid=5315 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.
```
Гуглим ошибку и натыкаемся на статью с похожей проблемой: https://bugzilla.redhat.com/show_bug.cgi?id=518749. 
Посмотрим SELinux context динамической директории:
```
[root@ns01 ~]# ls -dZ /etc/named/dynamic/
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   /etc/named/dynamic/
```
Видим тип etc_t, а должен быть named_cache_t (https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/sect-managing_confined_services-bind-configuration_examples) 

Поменяем контекст командами:
```
[root@ns01 ~]# semanage fcontext -a -t named_cache_t "/etc/named/dynamic(/.*)?"
[root@ns01 ~]# restorecon -R -v /etc/named/dynamic/
```
Еще раз проверим контекст:
```
[root@ns01 ~]# ls -dZ /etc/named/dynamic/
drw-rwx---. root named unconfined_u:object_r:named_cache_t:s0 /etc/named/dynamic/
```
Теперь контекст корректный.

Пробуем изменить зону:
```
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit

[vagrant@client ~]$ rndc -c ~/rndc.conf reload
server reload successful

[vagrant@client ~]$ dig @192.168.50.10 www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.10 <<>> @192.168.50.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 35357
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10

;; Query time: 0 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Tue Nov 15 14:07:00 UTC 2022
;; MSG SIZE  rcvd: 96
```
Зона успешно обновлена.

* Также для решения проблемы можно с помошью audit2allow создать разрешающий модуль, но данный способ не является оптимальным, потому что можно предоставить какому-либо приложению слишком широкие полномочия.
* Ещё одним способом решить проблему является полное отключение SELinux, но это также отрицательно скажется на безопасности системы.
