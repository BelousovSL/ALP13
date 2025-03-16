# Название выполняемого задания;
Практика с SELinux.

# Текст задания
Работать с SELinux: диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений, если это требуется;

## Часть 1.
Запустить Nginx на нестандартном порту 3-мя разными способами:
- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.

## Часть 2.
Обеспечить работоспособность приложения при включенном selinux.
- развернуть приложенный стенд https://github.com/Nickmob/vagrant_selinux_dns_problems; 
- выяснить причину неработоспособности механизма обновления зоны (см. README);
- предложить решение (или решения) для данной проблемы;
- выбрать одно из решений для реализации, предварительно обосновав выбор;
- реализовать выбранное решение и продемонстрировать его работоспособность.

# Реализация

## Часть 1.

Запускаем
```bash
cd part_1
vagrant up
```

При запуске, возникатие ошибка.
```text
    selinux: 
    selinux: Mar 16 17:17:27 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
    selinux: Mar 16 17:17:27 selinux nginx[6633]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    selinux: Mar 16 17:17:27 selinux nginx[6633]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
    selinux: Mar 16 17:17:27 selinux nginx[6633]: nginx: configuration file /etc/nginx/nginx.conf test failed
    selinux: Mar 16 17:17:27 selinux systemd[1]: nginx.service: Control process exited, code=exited, status=1/FAILURE
    selinux: Mar 16 17:17:27 selinux systemd[1]: nginx.service: Failed with result 'exit-code'.
    selinux: Mar 16 17:17:27 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
The SSH command responded with a non-zero exit status. Vagrant
assumes that this means the command failed. The output for this command
should be in the log above. Please read the output to determine what
went wrong.
```

Проблема очевидна, блокировка порта 4881.

Подключаемся к ВМ
```bash
ssh vagrant@127.0.0.1 -p 2222
```

### Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool

```bash
sudo -i
audit2why < /var/log/audit/audit.log
```
Получаем вывод
```text
type=AVC msg=audit(1742145447.104:771): avc:  denied  { name_bind } for  pid=6633 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
```

```bash
setsebool -P nis_enabled 1
```

Проверяем включение
```bash
[root@selinux ~]# getsebool -a | grep nis_enabled
nis_enabled --> on
```

Перезапускам nginx
```bash
systemctl restart nginx
```

Из хостовой машины шлем get запрос.
```bash
curl http://127.0.0.1:4881
```
Получаем отвтет
```bash
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.1//EN" "http://www.w3.org/TR/xhtml11/DTD/xhtml11.dtd">

<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en">
	<head>
		<title>Test Page for the HTTP Server on AlmaLinux</title>
		<meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
		<style type="text/css">
			/*<![CDATA[*/
			body {

...
```

Выключаем
```bash
setsebool -P nis_enabled off
```

Перезапускаем сервис Nginx и убеждаемся что он снова не работает

```bash
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.
```

### Разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип

```bash
[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

Добавляем наш порт.
```bash
semanage port -a -t http_port_t -p tcp 4881
```

Перезапускаем nginx
```bash
systemctl restart nginx
```

Проверяем и убеждаемся что все работает
```bash
sergey@fedora:~/Otus/homework/13/ALP13/part_1$ curl http://127.0.0.1:4881 
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.1//EN" "http://www.w3.org/TR/xhtml11/DTD/xhtml11.dtd">

<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en">
	<head>
		<title>Test Page for the HTTP Server on AlmaLinux</title>
		<meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
		<style type="text/css">
			/*<![CDATA[*/
			body {
				background-color: #FAF5F5;
				color: #000;
				font-size: 0.9em;
				font-family: sans-serif,helvetica;
				margin: 0;
				padding: 0;

```
Возвращаем все в "поломатое" состояние.

```bash
semanage port -d -t http_port_t -p tcp 4881
```

### Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux

```bash
[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.
```

```bash
[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp
```

Выполняем команду как нам рекомендуют
```bash
semodule -i nginx.pp
```

Перезапускаем nginx
```bash
systemctl restart nginx
```

Проверяем и убеждаемся что все работает
```bash
sergey@fedora:~/Otus/homework/13/ALP13/part_1$ curl http://127.0.0.1:4881
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.1//EN" "http://www.w3.org/TR/xhtml11/DTD/xhtml11.dtd">

<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en">
	<head>
		<title>Test Page for the HTTP Server on AlmaLinux</title>
		<meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
		<style type="text/css">
			/*<![CDATA[*/
			body {
				background-color: #FAF5F5;
				color: #000;
				font-size: 0.9em;
				font-family: sans-serif,helvetica;


```

## Часть 2.

Запускаем
```bash
cd part_2
vagrant up
```

Подключаемся к клиенту, убеждаемся что все плохо.

```bash
ssh vagrant@127.0.0.1 -p 2200
```

```bash
[vagrant@ns01 ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
```

Подключаемся к серверу
```bash
ssh vagrant@127.0.0.1 -p 2222
```

```bash
sudo -i
```

```bash
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1742149115.889:675): avc:  denied  { dac_read_search } for  pid=3320 comm="20-chrony-dhcp" capability=2  scontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tcontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tclass=capability permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1742149115.889:675): avc:  denied  { dac_override } for  pid=3320 comm="20-chrony-dhcp" capability=1  scontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tcontext=system_u:system_r:NetworkManager_dispatcher_chronyc_t:s0 tclass=capability permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1742149289.178:1782): avc:  denied  { write } for  pid=7226 comm="isc-net-0001" name="dynamic" dev="sda4" ino=34068225 scontext=system_u:system_r:named_t:s0 tcontext=unconfined_u:object_r:named_conf_t:s0 tclass=dir permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.
```
По логу становится понятно что проблема в named_conf_t

Проверяем
```bash
[root@ns01 ~]# ls -alZ /var/named/named.localhost
-rw-r-----. 1 root named system_u:object_r:named_zone_t:s0 152 Feb 19 16:04 /var/named/named.localhost
```

"named_zone_t" != "named_conf_t"

Меняем
```bash
chcon -R -t named_zone_t /etc/named
```

```bash
[root@ns01 ~]# ls -laZ /etc/named
total 28
drw-rwx---.  3 root named system_u:object_r:named_zone_t:s0      121 Mar 16 18:19 .
drwxr-xr-x. 85 root root  system_u:object_r:etc_t:s0            8192 Mar 16 18:19 ..
drw-rwx---.  2 root named unconfined_u:object_r:named_zone_t:s0   56 Mar 16 18:19 dynamic
-rw-rw----.  1 root named system_u:object_r:named_zone_t:s0      784 Mar 16 18:19 named.50.168.192.rev
-rw-rw----.  1 root named system_u:object_r:named_zone_t:s0      610 Mar 16 18:18 named.dns.lab
-rw-rw----.  1 root named system_u:object_r:named_zone_t:s0      609 Mar 16 18:19 named.dns.lab.view1
-rw-rw----.  1 root named system_u:object_r:named_zone_t:s0      657 Mar 16 18:19 named.newdns.lab
```

Проверяем на клиенте
```bash
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
```

Ошибки нет.

