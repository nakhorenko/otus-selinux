Начнём



<details>
  <summary>Часть 1</summary> 
  
Заходим на машину, проверяем статус firewalld и настройки nginx  

```
[root@selinux ~]# systemctl status firewalld.service 
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
Находим в логах (/var/log/audit/audit.log) информацию о блокировании порта
Копируем время, в которое был записан этот лог, и, с помощью утилиты audit2why смотрим
```
grep 1636489992.273:967 /var/log/audit/audit.log | audit2why
```
Тут оказалось, что audit2why в комплекте с центос 7 не входит

```
[root@selinux ~]# yum install policycoreutils-python
```
после
```
[root@selinux ~]# grep 1683288158.655:861 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1683288158.655:861): avc:  denied  { name_bind } for  pid=2979 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean nis_enabled was set incorrectly. 
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1
```
последние строки говорят нам, как можно это починить

```
[root@selinux ~]# setsebool -P nis_enabled 1
```
перезапускаем nginx, теперь nginx работает на порту 4881
```
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2023-05-05 12:23:19 UTC; 2s ago
  Process: 3296 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3294 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3293 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3298 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3298 nginx: master process /usr/sbin/nginx
           └─3300 nginx: worker process

May 05 12:23:19 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 05 12:23:19 selinux nginx[3294]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 05 12:23:19 selinux nginx[3294]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 05 12:23:19 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
проверка - работает
```
[root@selinux ~]# curl -I localhost:4881
HTTP/1.1 200 OK
Server: nginx/1.20.1
Date: Fri, 05 May 2023 12:28:06 GMT
Content-Type: text/html
Content-Length: 4833
Last-Modified: Fri, 16 May 2014 15:12:48 GMT
Connection: keep-alive
ETag: "53762af0-12e1"
Accept-Ranges: bytes

[root@selinux ~]# getsebool -a | grep nis_enabled
nis_enabled --> on
```
  
Вернём запрет работы nginx на порту 4881 обратно. Для этого отключим nis_enabled: 
```
[root@selinux ~]# setsebool -P nis_enabled off
```

Теперь разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип:
```
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
Добавим порт в тип http_port_t: 
```
[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep 4881     
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
```
Работает:
```
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2023-05-05 12:41:04 UTC; 1s ago
  Process: 3342 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3339 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3338 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3344 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3344 nginx: master process /usr/sbin/nginx
           └─3346 nginx: worker process

May 05 12:41:04 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
May 05 12:41:04 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 05 12:41:04 selinux nginx[3339]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 05 12:41:04 selinux nginx[3339]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 05 12:41:04 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
Удаляем нестандартный порт из имеющегося типа

```
[root@selinux ~]# semanage port -d -t http_port_t -p tcp 4881
[root@selinux ~]# systemctl restart nginx.service 
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]# systemctl status nginx.service 
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Fri 2023-05-05 12:44:34 UTC; 12s ago
  Process: 3342 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3394 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 3393 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3344 (code=exited, status=0/SUCCESS)

May 05 12:44:34 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
May 05 12:44:34 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 05 12:44:34 selinux nginx[3394]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 05 12:44:34 selinux nginx[3394]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
May 05 12:44:34 selinux nginx[3394]: nginx: configuration file /etc/nginx/nginx.conf test failed
May 05 12:44:34 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
May 05 12:44:34 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
May 05 12:44:34 selinux systemd[1]: Unit nginx.service entered failed state.
May 05 12:44:34 selinux systemd[1]: nginx.service failed.
```
Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux
```
[root@selinux ~]# grep nginx /var/log/audit/audit.log
type=SYSCALL msg=audit(1683290887.132:942): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=558cf9298878 a2=10 a3=7ffca811b960 items=0 ppid=1 pid=3407 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1683290887.132:943): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
```
Воспользуемся утилитой audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту: 
```
[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
```
Нам подсказывают как применить сформированный системой модуль. Выполняем и запускаем вебсервер
```
[root@selinux ~]# semodule -i nginx.pp
[root@selinux ~]# systemctl start nginx
[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2023-05-05 12:55:20 UTC; 4s ago
  Process: 3437 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3435 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 3434 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3439 (nginx)
   CGroup: /system.slice/nginx.service
           ├─3439 nginx: master process /usr/sbin/nginx
           └─3441 nginx: worker process

May 05 12:55:20 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 05 12:55:20 selinux nginx[3435]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 05 12:55:20 selinux nginx[3435]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 05 12:55:20 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
</details>
