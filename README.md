# Практика с SELinux

## Задание

Обеспечить работоспособность приложения при включенном **SELinux**.

- развернуть приложенный стенд [selinux_dns_problems](https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems);
- выяснить причину неработоспособности механизма обновления зоны (см. README);
- предложить решение (или решения) для данной проблемы;
- выбрать одно из решений для реализации, предварительно обосновав выбор;
- реализовать выбранное решение и продемонстрировать его работоспособность.

## Реализация

Решение находится в файле **[selinux.yml](selinux.yml)**.

Стенд [selinux_dns_problems](https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems) переделан:

- переделан **[Vagrantfile](Vagrantfile)**;
- **centos/7"** заменён на **almalinux/9/v9.4.20240805**;
- **[playbook.yml](playbook.yml)** разбит на **[all.yml](all.yml)**, **[ns01.yml](ns01.yml)** и **[client.yml](client.yml)**;
- исправлены ошибки линтера и проблемы с установкой **ntp**.

## Отладка

При подключении к **client** видно, что проблема в описании воспроизводится на **almalinux/9**:

```text
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
```

Из ошибки **SERVFAIL** понятно, что проблема возникает на сервере **ns01**. Подключимся к **ns01** и посмотрим логи **bind** командой **journalctl -u named**:

```text
Sep 07 14:44:19 ns01.internal named[12633]: client @0x7f455c86c678 192.168.50.15#35960/key zonetransfer.key: view view1: updating zone 'ddns.lab/IN': adding an RR at 'www.ddns.lab' A 192.168.50.15
Sep 07 14:44:19 ns01.internal named[12633]: /etc/named/dynamic/named.ddns.lab.view1.jnl: create: permission denied
Sep 07 14:44:19 ns01.internal named[12633]: client @0x7f455c86c678 192.168.50.15#35960/key zonetransfer.key: view view1: updating zone 'ddns.lab/IN': error: journal open failed: unexpected error
```

Из ошибки видно, что отсутствуют права на **/etc/named/dynamic/named.ddns.lab.view1.jnl**. Посмотрим права на файлы в этой директории:

```text
[root@ns01 ~]# ls -la /etc/named/dynamic/
total 8
drw-rwx---. 2 root  named  56 Sep  7 14:41 .
drw-rwx---. 3 root  named 121 Sep  7 14:41 ..
-rw-rw----. 1 named named 509 Sep  7 14:41 named.ddns.lab
-rw-rw----. 1 named named 509 Sep  7 14:41 named.ddns.lab.view1
```

У **named** есть все необходимые права. Проверим журнал аудита:

```text
[root@ns01 ~]# sealert --analyze /var/log/audit/audit.log
100% done
found 3 alerts in /var/log/audit/audit.log
...
SELinux is preventing /usr/sbin/named from write access on the directory dynamic.

*****  Plugin catchall (100. confidence) suggests   **************************

If you believe that named should be allowed write access on the dynamic directory by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c 'isc-net-0000' --raw | audit2allow -M my-iscnet0000
# semodule -X 300 -i my-iscnet0000.pp


Additional Information:
Source Context                system_u:system_r:named_t:s0
Target Context                unconfined_u:object_r:named_conf_t:s0
Target Objects                dynamic [ dir ]
Source                        isc-net-0000
Source Path                   /usr/sbin/named
Port                          <Unknown>
Host                          <Unknown>
Source RPM Packages           bind-9.16.23-18.el9_4.6.x86_64
Target RPM Packages
SELinux Policy RPM            selinux-policy-targeted-38.1.35-2.el9_4.2.noarch
Local Policy RPM              selinux-policy-targeted-38.1.35-2.el9_4.2.noarch
Selinux Enabled               True
Policy Type                   targeted
Enforcing Mode                Enforcing
Host Name                     ns01.internal
Platform                      Linux ns01.internal 5.14.0-427.28.1.el9_4.x86_64
                              #1 SMP PREEMPT_DYNAMIC Fri Aug 2 03:44:10 EDT 2024
                              x86_64 x86_64
Alert Count                   1
First Seen                    2024-09-07 14:44:19 UTC
Last Seen                     2024-09-07 14:44:19 UTC
Local ID                      38a652e1-17c5-40d0-a756-63bfba8feeee

Raw Audit Messages
type=AVC msg=audit(1725720259.625:2531): avc:  denied  { write } for  pid=12633 comm="isc-net-0000" name="dynamic" dev="sda4" ino=33987691 scontext=system_u:system_r:named_t:s0 tcontext=unconfined_u:object_r:named_conf_t:s0 tclass=dir permissive=0


type=SYSCALL msg=audit(1725720259.625:2531): arch=x86_64 syscall=openat success=no exit=EACCES a0=ffffff9c a1=7f4560bfa050 a2=241 a3=1b6 items=0 ppid=1 pid=12633 auid=4294967295 uid=25 gid=25 euid=25 suid=25 fsuid=25 egid=25 sgid=25 fsgid=25 tty=(none) ses=4294967295 comm=isc-net-0000 exe=/usr/sbin/named subj=system_u:system_r:named_t:s0 key=(null)ARCH=x86_64 SYSCALL=openat AUID=unset UID=named GID=named EUID=named SUID=named FSUID=named EGID=named SGID=named FSGID=named

Hash: isc-net-0000,named_t,named_conf_t,dir,write
```

Из лога удалены ошибки **chrony**, так как они нас не интересуют. Из лога видно, что **isc-net-0000** в домене **named_t** пытается писать в директорию с типом **named_conf_t**. Поищем в существующих политиках директории, куда он может писать:

```text
root@ns01 ~]# sesearch --allow --source named_t --class dir --perms write
allow daemon cluster_conf_t:dir { add_name create getattr ioctl link lock open read remove_name rename reparent rmdir search setattr unlink watch watch_reads write }; [ daemons_enable_cluster_mode ]:True
allow daemon cluster_conf_t:dir { add_name getattr ioctl lock open read remove_name search write }; [ daemons_enable_cluster_mode ]:True
allow daemon cluster_conf_t:dir { add_name getattr ioctl lock open read remove_name search write }; [ daemons_enable_cluster_mode ]:True
allow daemon cluster_var_lib_t:dir { add_name getattr ioctl lock open read remove_name search write }; [ daemons_enable_cluster_mode ]:True
allow daemon cluster_var_run_t:dir { add_name ioctl lock read remove_name write }; [ daemons_enable_cluster_mode ]:True
allow daemon root_t:dir { add_name remove_name write }; [ daemons_dump_core ]:True
allow domain tmpfs_t:dir { add_name getattr ioctl lock open read remove_name search write };
allow named_t admin_home_t:dir { add_name ioctl lock read remove_name write };
allow named_t dnssec_trigger_var_run_t:dir { add_name create ioctl link lock read remove_name rename reparent rmdir setattr unlink watch watch_reads write };
allow named_t etc_t:dir { add_name remove_name write };
allow named_t krb5kdc_conf_t:dir { add_name getattr ioctl lock open read remove_name search write };
allow named_t named_cache_t:dir { add_name create getattr ioctl link lock open read remove_name rename reparent rmdir search setattr unlink watch watch_reads write };
allow named_t named_log_t:dir { add_name getattr ioctl lock open read remove_name search write };
allow named_t named_tmp_t:dir { add_name create getattr ioctl link lock open read remove_name rename reparent rmdir search setattr unlink watch watch_reads write };
allow named_t named_var_run_t:dir { add_name create ioctl link lock read remove_name rename reparent rmdir setattr unlink watch watch_reads write };
allow named_t named_zone_t:dir { add_name create link remove_name rename reparent rmdir setattr unlink watch watch_reads write }; [ named_write_master_zones ]:True
allow named_t named_zone_t:dir { add_name remove_name write }; [ named_write_master_zones ]:True
allow named_t named_zone_t:dir { add_name remove_name write }; [ named_write_master_zones ]:True
allow named_t named_zone_t:dir { add_name remove_name write }; [ named_write_master_zones ]:True
allow named_t var_log_t:dir { add_name ioctl lock read remove_name write };
allow named_t var_run_t:dir { add_name remove_name write };
allow nsswitch_domain krb5_host_rcache_t:dir { add_name getattr ioctl lock open read remove_name search write };
allow nsswitch_domain tmp_t:dir { add_name ioctl lock read remove_name write };
```

Из лога видно, что домену **named_t** не разрешено писать в директории с типом **named_conf_t**, но можно писать в **named_zone_t**, при включённой политике **named_write_master_zones**. Поищем в политике директории с типом **named_zone_t**:

```text
[root@ns01 ~]# semanage fcontext --list | grep '\bnamed_zone_t\b'
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0
/var/named/chroot/var/named(/.*)?                  all files          system_u:object_r:named_zone_t:s0
```

Соответственно, проблема в том, что директория **/etc/named/dynamic** находится в директории **/etc/named** вместо **/var/named**. Мы можем либо её переместить в правильное расположение и переконфигурировать сервер или добавать в политику неправильное расположение. Поскольку это задание по **SELinux**, а не настройке **bind**, то будем использовать второе решение:

```shell
semanage fcontext --add --type named_zone_t '/etc/named/dynamic(/.*)?'
restorecon -r /etc/named/dynamic
```

Проверяем добавление:

```text
[root@ns01 ~]# cat /etc/selinux/targeted/contexts/files/file_contexts.local
# This file is auto-generated by libsemanage
# Do not edit directly.

/etc/named/dynamic(/.*)?    system_u:object_r:named_zone_t:s0
[root@ns01 ~]# semanage fcontext --list | grep '\bnamed_zone_t\b'
/etc/named/dynamic(/.*)?                           all files          system_u:object_r:named_zone_t:s0
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0
/var/named/chroot/var/named(/.*)?                  all files          system_u:object_r:named_zone_t:s0
[root@ns01 ~]# ls -laZ /etc/named/dynamic
total 8
drw-rwx---. 2 root  named unconfined_u:object_r:named_zone_t:s0  56 Sep  7 14:41 .
drw-rwx---. 3 root  named system_u:object_r:named_conf_t:s0     121 Sep  7 14:41 ..
-rw-rw----. 1 named named system_u:object_r:named_zone_t:s0     509 Sep  7 14:41 named.ddns.lab
-rw-rw----. 1 named named system_u:object_r:named_zone_t:s0     509 Sep  7 14:41 named.ddns.lab.view1
```

Нам нужно также убедиться, что включена политика **named_write_master_zones**:

```text
[root@ns01 ~]# getsebool named_write_master_zones
named_write_master_zones --> on
```

Политика уже включена и никакие действия не требуются. Проверяем работу:

```text
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> [vagrant@client ~]dig @192.168.50.10 ns01.dns.labab

; <<>> DiG 9.16.23-RH <<>> @192.168.50.10 ns01.dns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 57320
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: ca40a61beb5ec5310100000066dc72eb9ee005edd0bea6fa (good)
;; QUESTION SECTION:
;ns01.dns.lab.                  IN      A

;; ANSWER SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10

;; Query time: 0 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Sat Sep 07 15:36:11 UTC 2024
;; MSG SIZE  rcvd: 85
```

Теперь оформим решение в **[selinux.yml](selinux.yml)**.

## Запуск

Необходимо скачать **VagrantBox** для **almalinux/9** версии **v9.4.20240805** и добавить его в **Vagrant** под именем **almalinux/9/v9.4.20240805**. Сделать это можно командами:

```shell
curl -OL https://app.vagrantup.com/almalinux/boxes/9/versions/9.4.20240805/providers/virtualbox/amd64/vagrant.box
vagrant box add vagrant.box --name "almalinux/9/v9.4.20240805"
rm vagrant.box
```

После этого нужно сделать **vagrant up**.

Протестировано в **OpenSUSE Tumbleweed**:

- **Vagrant 2.3.7**
- **VirtualBox 7.0.20_SUSE r163906**
- **Ansible 2.17.3**
- **Python 3.11.9**
- **Jinja2 3.1.4**
