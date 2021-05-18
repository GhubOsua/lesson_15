# Урок 15. SELinux - когда все запрещено 

## 1. Задание №1. Устанавливаем пакеты для управления selinux. Плюс устанавливаем nginx и правим конфигурацию nginx;

### 1.1 yum install policycoreutils-python setroubleshoot setools-console epel-release -y;
yum install -y nginx
Стандартные порты в selinux для nginx, след.:
```
root@lesson15 ~]# semanage port -l | grep http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
```

### 1.2 После правим конфигурацию nginx, vi /etc/nginx/nginx.conf. Меняем порт с 80 на 
. После перезапуска nginx, ssytemd сообщает об ошибке;
```
[root@lesson15 ~]# systemctl status  nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Sun 2021-05-16 10:14:47 UTC; 7s ago
  Process: 3302 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 3418 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 3416 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 3304 (code=exited, status=0/SUCCESS)

May 16 10:14:47 lesson15 systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 16 10:14:47 lesson15 nginx[3418]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 16 10:14:47 lesson15 nginx[3418]: nginx: [emerg] bind() to 0.0.0.0:6520 failed (13: Permission denied)
May 16 10:14:47 lesson15 nginx[3418]: nginx: configuration file /etc/nginx/nginx.conf test failed
May 16 10:14:47 lesson15 systemd[1]: nginx.service: control process exited, code=exited status=1
May 16 10:14:47 lesson15 systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
May 16 10:14:47 lesson15 systemd[1]: Unit nginx.service entered failed state.
May 16 10:14:47 lesson15 systemd[1]: nginx.service failed.
```

### 1.3 Для анализа ошибки используем утилиту sealert;
```
[root@lesson15 ~]# sealert -a /var/log/audit/audit.log 
```

## 2. Задание №1 Запустить nginx на нестандартном порту 6520 3-мя разными способами:
### 2.1 Способ "переключатели setsebool";
Из вывода пункта 1.3, получается необходимо изменить параметризованные политики SELinux:
```
*****  Plugin catchall_boolean (7.83 confidence) suggests   ******************

If you want to allow nis to enabled
Then you must tell SELinux about this by enabling the 'nis_enabled' boolean.

Do
setsebool -P nis_enabled 1

```
Выполняем и получаем работающий nginx:
```
[root@lesson15 ~]# setsebool -P nis_enabled 1
[root@lesson15 ~]# systemctl restart nginx
[root@lesson15 ~]# systemctl status  nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2021-05-16 10:43:32 UTC; 4s ago
  Process: 6347 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 6344 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 6342 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 6349 (nginx)
   CGroup: /system.slice/nginx.service
           ├─6349 nginx: master process /usr/sbin/nginx
           ├─6350 nginx: worker process
           └─6351 nginx: worker process

May 16 10:43:32 lesson15 systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 16 10:43:32 lesson15 nginx[6344]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 16 10:43:32 lesson15 nginx[6344]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 16 10:43:32 lesson15 systemd[1]: Started The nginx HTTP and reverse proxy server.
```

### 2.2 Способ "добавление нестандартного порта в имеющийся тип;"
Из вывода пункта 1.3, получается необходимо изменить тип порта SELinux:
```
*****  Plugin bind_ports (92.2 confidence) suggests   ************************

If you want to allow /usr/sbin/nginx to bind to network port 6520
Then you need to modify the port type.
Do
# semanage port -a -t PORT_TYPE -p tcp 6520
    where PORT_TYPE is one of the following: http_cache_port_t, http_port_t, jboss_management_port_t, jboss_messaging_port_t, ntop_port_t, puppet_port_t.
```

Выполняем команду, только за место PORT_TYPE набираем http_port_t потому что это порт тип nginx.
```
[root@lesson15 ~]# semanage port -l | grep http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@lesson15 ~]# semanage port -a -t http_port_t -p tcp 6520
[root@lesson15 ~]# semanage port -l | grep http_port_t
http_port_t                    tcp      6520, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988

[root@lesson15 ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2021-05-16 11:06:57 UTC; 1s ago
  Process: 6695 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 6691 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 6689 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 6697 (nginx)
   CGroup: /system.slice/nginx.service
           ├─6697 nginx: master process /usr/sbin/nginx
           ├─6698 nginx: worker process
           └─6699 nginx: worker process

May 16 11:06:57 lesson15 systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 16 11:06:57 lesson15 nginx[6691]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 16 11:06:57 lesson15 nginx[6691]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 16 11:06:57 lesson15 systemd[1]: Started The nginx HTTP and reverse proxy server.
```

### 2.3 Формирование и установка модуля SELinux;
Из вывода пункта 1.3, получается необходимо изменить тип порта SELinux:
```
*****  Plugin catchall (1.41 confidence) suggests   **************************

If you believe that nginx should be allowed name_bind access on the port 6530 tcp_socket by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
# semodule -i my-nginx.pp
```
Выполняем след. команды:
```
[root@lesson15 ~]# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i my-nginx.pp

[root@lesson15 ~]# semodule -i my-nginx.pp
[root@lesson15 ~]# semodule -l | grep nginx
my-nginx	1.0
[root@lesson15 ~]# systemctl restart nginx
[root@lesson15 ~]# systemctl status  nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2021-05-16 11:36:54 UTC; 6s ago
  Process: 6931 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 6928 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 6926 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 6933 (nginx)
   CGroup: /system.slice/nginx.service
           ├─6933 nginx: master process /usr/sbin/nginx
           ├─6934 nginx: worker process
           └─6935 nginx: worker process

May 16 11:36:54 lesson15 systemd[1]: Starting The nginx HTTP and reverse proxy server...
May 16 11:36:54 lesson15 nginx[6928]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
May 16 11:36:54 lesson15 nginx[6928]: nginx: configuration file /etc/nginx/nginx.conf test is successful
May 16 11:36:54 lesson15 systemd[1]: Started The nginx HTTP and reverse proxy server.
```

## 3. Задание №2. "Развернуть приложенный стенд и Выяснить причину неработоспособности механизма обновления зоны";
### 3.1 Клонируем репозиторий https://github.com/mbfx/otus-linux-adm. Запускаем vagrant машины;
```
https://github.com/mbfx/otus-linux-adm

```
Далее мы подключаемся к ns01 и к client. И на client выполняем указанную в README команду nsupdate -k /etc/named.zonetransfer.key. Появляется ошибка update failed: SERVFAIL.

### 3.2 Начинаем изучать ошибку на ns01. Т.к. нам sealert -a толком не подсказывает, выполняем др. команду audit2why < /var/log/audit/audit.log;
```
[root@ns01 ~]# audit2why < /var/log/audit/audit.log 
type=AVC msg=audit(1621187587.507:2163): avc:  denied  { create } for  pid=5497 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

[root@ns01 ~]# sealert -a /var/log/audit/audit.log 
100% done
found 1 alerts in /var/log/audit/audit.log
--------------------------------------------------------------------------------
```
Нас просят сгенерировать и загрузить модуль named. В логах /var/log/messages есть готовая команда
```
#012Then you should report this as a bug.#012You can generate a local policy module to allow this access.#012Do#012allow this access for now by executing:#012# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000#012# semodule -i my-iscworker0000.pp#012
```

### 3.3 Формируем модуль и загружаем его в SELinux. Модуль my-iscworker0000.pp;
```
[root@ns01 ~]# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i my-iscworker0000.pp

[root@ns01 ~]# semodule -i my-iscworker0000.pp
[root@ns01 ~]# semodule -l | grep my
my-iscworker0000	1.0
mysql	1.14.1
mythtv	1.0.0
```

Получаем ошибку:

```
May 16 18:25:59 localhost setroubleshoot: Deleting alert 239ab45d-423a-4ea3-b62f-525216019894, it is allowed in current policy
May 16 18:25:59 localhost setroubleshoot: SELinux is preventing isc-worker0000 from write access on the file /etc/named/dynamic/named.ddns.lab.view1.jnl. For complete SELinux messages run: sealert -l fc17b420-edd1-442d-a903-efa833945fa5
May 16 18:25:59 localhost python: SELinux is preventing isc-worker0000 from write access on the file /etc/named/dynamic/named.ddns.lab.view1.jnl.#012#012*****  Plugin catchall_labels (83.8 confidence) suggests   *******************#012#012If you want to allow isc-worker0000 to have write access on the named.ddns.lab.view1.jnl file#012Then you need to change the label on /etc/named/dynamic/named.ddns.lab.view1.jnl#012Do#012# semanage fcontext -a -t FILE_TYPE '/etc/named/dynamic/named.ddns.lab.view1.jnl'#012where FILE_TYPE is one of the following: afs_cache_t, dnssec_trigger_var_run_t, initrc_tmp_t, ipa_var_lib_t, krb5_host_rcache_t, krb5_keytab_t, named_cache_t, named_log_t, named_tmp_t, named_var_run_t, named_zone_t, puppet_tmp_t, user_cron_spool_t, user_tmp_t.#012Then execute:#012restorecon -v '/etc/named/dynamic/named.ddns.lab.view1.jnl'#012#012#012*****  Plugin catchall (17.1 confidence) suggests   **************************#012#012If you believe that isc-worker0000 should be allowed write access on the named.ddns.lab.view1.jnl file by default.#012Then you should report this as a bug.#012You can generate a local policy module to allow this access.#012Do#012allow this access for now by executing:#012# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000#012# semodule -i my-iscworker0000.pp#012

```
### 3.4 Изменяем контекcт тип для файла '/etc/named/dynamic/named.ddns.lab.view1.jnl';
После изменения контекста, у меня больше не появлялись ошибки с selinux и не работала динамическое обновление dns. После удаления файла named.ddns.lab.view1.jnl, снова появилась ошибка 
/etc/named/dynamic/named.ddns.lab.view1.jnl: create: permission denied. 
```
[root@ns01 ~]# semanage fcontext -a -t named_cache_t '/etc/named/dynamic/named.ddns.lab.view1.jnl' 
[root@ns01 ~]# ll -Z /etc/named/dynamic/
-rw-rw----. named named system_u:object_r:etc_t:s0       named.ddns.lab
-rw-rw----. named named system_u:object_r:etc_t:s0       named.ddns.lab.view1
-rw-r--r--. named named system_u:object_r:etc_t:s0       named.ddns.lab.view1.jnl
[root@ns01 ~]# restorecon -R -v /etc/named/dynamic/named.ddns.lab.view1.jnl
restorecon reset /etc/named/dynamic/named.ddns.lab.view1.jnl context system_u:object_r:etc_t:s0->system_u:object_r:named_cache_t:s0
[root@ns01 ~]# ll -Z /etc/named/dynamic/
-rw-rw----. named named system_u:object_r:etc_t:s0       named.ddns.lab
-rw-rw----. named named system_u:object_r:etc_t:s0       named.ddns.lab.view1
-rw-r--r--. named named system_u:object_r:named_cache_t:s0 named.ddns.lab.view1.jnl
```
В службе нет ошибок:
```
[root@ns01 ~]# systemctl status named.service 
● named.service - Berkeley Internet Name Domain (DNS)
   Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2021-05-16 18:46:53 UTC; 4s ago
  Process: 26175 ExecStop=/bin/sh -c /usr/sbin/rndc stop > /dev/null 2>&1 || /bin/kill -TERM $MAINPID (code=exited, status=0/SUCCESS)
  Process: 26188 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)
  Process: 26186 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-checkconf -z "$NAMEDCONF"; else echo "Checking of zone files is disabled"; fi (code=exited, status=0/SUCCESS)
 Main PID: 26190 (named)
   CGroup: /system.slice/named.service
           └─26190 /usr/sbin/named -u named -c /etc/named.conf

May 16 18:46:53 ns01 named[26190]: network unreachable resolving './NS/IN': 2001:500:200::b#53
May 16 18:46:53 ns01 named[26190]: managed-keys-zone/view1: Key 20326 for zone . acceptance timer complete: key now trusted
May 16 18:46:53 ns01 named[26190]: resolver priming query complete
May 16 18:46:53 ns01 named[26190]: network unreachable resolving './DNSKEY/IN': 2001:500:1::53#53
May 16 18:46:53 ns01 named[26190]: network unreachable resolving './DNSKEY/IN': 2001:7fd::1#53
May 16 18:46:53 ns01 named[26190]: network unreachable resolving './DNSKEY/IN': 2001:7fe::53#53
May 16 18:46:53 ns01 named[26190]: network unreachable resolving './DNSKEY/IN': 2001:500:200::b#53
May 16 18:46:53 ns01 named[26190]: network unreachable resolving './DNSKEY/IN': 2001:dc3::35#53
May 16 18:46:53 ns01 named[26190]: managed-keys-zone/default: Key 20326 for zone . acceptance timer complete: key now trusted
May 16 18:46:53 ns01 named[26190]: resolver priming query complete
```
Но не проходит данная операция, тогда использую audit2allow:
```
[root@ns01 ~]# audit2why < /var/log/audit/audit.log 
type=AVC msg=audit(1621187587.507:2163): avc:  denied  { create } for  pid=5497 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Unknown - would be allowed by active policy
		Possible mismatch between this policy and the one under which the audit message was generated.

		Possible mismatch between current in-memory boolean settings vs. permanent ones.

type=AVC msg=audit(1621189555.825:2173): avc:  denied  { write } for  pid=5497 comm="isc-worker0000" path="/etc/named/dynamic/named.ddns.lab.view1.jnl" dev="sda1" ino=100672908 scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.
```

### 3.5 Удаляем файл /etc/named/dynamic/named.ddns.lab.view1.jnl: create: permission denied и формируем модуль my-iscworker0001, чтобы создался файл /etc/named/dynamic/named.ddns.lab.view1.jnl;
Удалил файл /etc/named/dynamic/named.ddns.lab.view1.jnl
После удалания файла named.ddns.lab.view1.jnl, снова появилась ошибка 
/etc/named/dynamic/named.ddns.lab.view1.jnl: create: permission denied. 
Selinux сообщил:
```
[root@ns01 ~]# sealert -a /var/log/audit/audit.log 
100% done
found 1 alerts in /var/log/audit/audit.log
--------------------------------------------------------------------------------

Traceback (most recent call last):
  File "/bin/sealert", line 510, in on_analyzer_state_change
    self.output_results()
  File "/bin/sealert", line 524, in output_results
    print siginfo.format_text()
UnicodeEncodeError: 'ascii' codec can't encode characters in position 8-16: ordinal not in range(128)
```

```
[root@ns01 ~]# audit2why < /var/log/audit/audit.log
type=AVC msg=audit(1621358549.227:759): avc:  denied  { create } for  pid=2719 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1621359761.265:762): avc:  denied  { create } for  pid=2719 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

```

В messages ошибка:
```
May 18 17:42:44 ns01 setroubleshoot: SELinux is preventing isc-worker0000 from create access on the file named.ddns.lab.view1.jnl. For complete SELinux messages run: sealert -l f5f71a93-2434-448f-8db3-dd5b7920ae6f
May 18 17:42:44 ns01 python: SELinux is preventing isc-worker0000 from create access on the file named.ddns.lab.view1.jnl.#012#012*****  Plugin catchall_labels (83.8 confidence) suggests   *******************#012#012If you want to allow isc-worker0000 to have create access on the named.ddns.lab.view1.jnl file#012Then you need to change the label on named.ddns.lab.view1.jnl#012Do#012# semanage fcontext -a -t FILE_TYPE 'named.ddns.lab.view1.jnl'#012where FILE_TYPE is one of the following: dnssec_trigger_var_run_t, ipa_var_lib_t, krb5_host_rcache_t, krb5_keytab_t, named_cache_t, named_log_t, named_tmp_t, named_var_run_t, named_zone_t.#012Then execute:#012restorecon -v 'named.ddns.lab.view1.jnl'#012#012#012*****  Plugin catchall (17.1 confidence) suggests   **************************#012#012If you believe that isc-worker0000 should be allowed create access on the named.ddns.lab.view1.jnl file by default.#012Then you should report this as a bug.#012You can generate a local policy module to allow this access.#012Do#012allow this access for now by executing:#012# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0000#012# semodule -i my-iscworker0000.pp#012

```
Далее выполняю команду ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0001 &&  semodule -i my-iscworker0001.pp , чтобы можно было создать файл named.ddns.lab.view1.jnl
```
[root@ns01 ~]# ausearch -c 'isc-worker0000' --raw | audit2allow -M my-iscworker0001 &&  semodule -i my-iscworker0001.pp
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i my-iscworker0001.pp

```
После обновляю dns и обновление проходит:
```
[root@ns01 ~]# systemctl status  named
● named.service - Berkeley Internet Name Domain (DNS)
   Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2021-05-18 18:34:42 UTC; 3min 38s ago
  Process: 3274 ExecStop=/bin/sh -c /usr/sbin/rndc stop > /dev/null 2>&1 || /bin/kill -TERM $MAINPID (code=exited, status=0/SUCCESS)
  Process: 3286 ExecStart=/usr/sbin/named -u named -c ${NAMEDCONF} $OPTIONS (code=exited, status=0/SUCCESS)
  Process: 3284 ExecStartPre=/bin/bash -c if [ ! "$DISABLE_ZONE_CHECKING" == "yes" ]; then /usr/sbin/named-checkconf -z "$NAMEDCONF"; else echo "Checking of zone files is disabled"; fi (code=exited, status=0/SUCCESS)
 Main PID: 3289 (named)
   CGroup: /system.slice/named.service
           └─3289 /usr/sbin/named -u named -c /etc/named.conf

May 18 18:34:42 ns01 named[3289]: managed-keys-zone/default: journal file is out of date: removing journal file
May 18 18:34:42 ns01 named[3289]: managed-keys-zone/default: loaded serial 24
May 18 18:34:42 ns01 systemd[1]: Started Berkeley Internet Name Domain (DNS).
May 18 18:35:20 ns01 named[3289]: client @0x7fb2d003c3e0 192.168.50.15#43114/key zonetransfer.key: view view1: signer "zonetransfer.key" approved
May 18 18:35:20 ns01 named[3289]: client @0x7fb2d003c3e0 192.168.50.15#43114/key zonetransfer.key: view view1: updating zone 'ddns.lab/IN': ad....168.50.15
May 18 18:35:20 ns01 named[3289]: /etc/named/dynamic/named.ddns.lab.view1.jnl: create: permission denied
May 18 18:35:20 ns01 named[3289]: client @0x7fb2d003c3e0 192.168.50.15#43114/key zonetransfer.key: view view1: updating zone 'ddns.lab/IN': er...cted error
May 18 18:37:36 ns01 named[3289]: client @0x7fb2d003c3e0 192.168.50.15#43114/key zonetransfer.key: view view1: signer "zonetransfer.key" approved
May 18 18:38:05 ns01 named[3289]: client @0x7fb2d003c3e0 192.168.50.15#43114/key zonetransfer.key: view view1: signer "zonetransfer.key" approved
May 18 18:38:05 ns01 named[3289]: client @0x7fb2d003c3e0 192.168.50.15#43114/key zonetransfer.key: view view1: updating zone 'ddns.lab/IN': ad....168.50.15

