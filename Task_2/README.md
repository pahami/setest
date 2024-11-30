1. Дистрибутив с Centos/7 устанавливается с устаревшими репозиториями. Добавил в плейбук команду по обновлению репозиториев для всех машин пере установкой необходимых пакетов.

- name: update repo
    shell: |
      repo_file=/etc/yum.repos.d/CentOS-Base.repo
      cp ${repo_file} ~/CentOS-Base.repo.backup
      sudo sed -i s/#baseurl/baseurl/ ${repo_file}
      sudo sed -i s/mirrorlist.centos.org/vault.centos.org/ ${repo_file}
      sudo sed -i s/mirror.centos.org/vault.centos.org/ ${repo_file}
      sudo yum clean all

P.S. Изначально были проблемы с разворачиванием Дистрибутивов. При создании не применялись сетевые настройки, не заупскался сразу playbook. Но ВМ создавалась и при повторном запуске vagrant up или provision запускался playbook. Проблема ушла, когда отключил cloud-init


2. На основе Vagrantfile создаётся 2 ВМ ns01 (dns server & client)

Проверяем работу

vagrant ssh client
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit

Проверяем ошибки на client
sudo -i
cat /var/log/audit/audit.log | audit2why

Видим, что ошибок нет. подключаемся к ns01 во втором окне терминала 

Для диагностики проблемы 
cat /var/log/audit/audit.log | audit2why

type=AVC msg=audit(1637070345.890:1972): avc:  denied  { create } for  pid=5192 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0


    Was caused by:
        Missing type enforcement (TE) allow rule.


        You can use audit2allow to generate a loadable module to allow this access.

Видим, чо ошибка в контексте безопасности. Вместо типа named_t в папке назначения используется тип etc_t. Это произошло в связи с тем, что файлы конфигурации находятся не в стандартном месте.

ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab

2 способа решения проблемы:

****************1. Создание локальных политик******************

cd /etc/named

Выполним audit2allow -w -a, чтобы подробнее изучить проблему

-w  понятное предстваление ошибки
-a  чтение всего журнала

audit2allow -w -a

type=AVC msg=audit(1732968056.430:2068): avc:  denied  { create } for  pid=5400 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0
        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

Видим, что запрещен доступ на создание в каталог, помеченный типом etc_t. Т.к. нет булевых значений, разрешающих доступ, используем audit2allow для создания модуля локальной политики

Запускаем audit2allow -a, чтобы просмотреть правило Type Enforcement, разрешающее запрещенный доступ:
audit2allow -a

#============= named_t ==============

#!!!! WARNING: 'etc_t' is a base type.
allow named_t etc_t:file create;

Выполним команду для создания пользовательского модуля. audit2allow -a -M named

 -M   Опция создает файл Type Enforcement (.te) с именем, указанным в нашем текущем рабочем каталоге

audit2allow -a -M named

******************** IMPORTANT ***********************
To make this policy package active, execute:

в каталоге создалось 2 файла

[root@ns01 named]# ls -al
total 36
drw-rwx---.  3 root named  153 Nov 30 12:09 .
drwxr-xr-x. 86 root root  8192 Nov 30 11:22 ..
drw-rwx---.  2 root named   88 Nov 30 12:08 dynamic
-rw-rw----.  1 root named  784 Nov 30 11:22 named.50.168.192.rev
-rw-rw----.  1 root named  610 Nov 30 11:22 named.dns.lab
-rw-rw----.  1 root named  609 Nov 30 11:22 named.dns.lab.view1
-rw-rw----.  1 root named  657 Nov 30 11:22 named.newdns.lab
-rw-r--r--.  1 root root  1034 Nov 30 12:09 named.pp
-rw-r--r--.  1 root root   243 Nov 30 12:09 named.te

semodule -i named.pp

Применим новую политику

semodule -i named.pp

**************2. Исправим контекст*************

Каталогу /etc/named присвоен тип etc_t, он унаследован от папки /etc. Посмотрим в каком каталоге должные лежать файлы 

sudo semanage fcontext -l | grep named

sudo semanage fcontext -l | grep named
/etc/rndc.*              regular file       system_u:object_r:named_conf_t:s0 
/var/named(/.*)?         all files          system_u:object_r:named_zone_t:s0 
...

Видим, что для каталога по умолчанию задан тип контекста named_zone_t

Изменим тип контекста для каталога /etc/named

chcon -R -t named_zone_t /etc/named
-R    изменения всех файлов в каталоге
-t    указание типа

Проверка:

vagrant ssh client
nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit 

dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.16 <<>> @192.168.50.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 18357
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; AUTHORITY SECTION:
ddns.lab.               600     IN      SOA     ns01.dns.lab. root.dns.lab. 2711201407 3600 600 86400 600

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Sat Nov 30 12:11:11 UTC 2024
;; MSG SIZE  rcvd: 91

********Выбор способа решения***********

Я выбрал второй способ решения "Исправление контекста".
etc_t является общим типом и при добавлении резрешающей политики модуля named.pp, он получил доступ ко всем каталогам данного типа, что не безопасно.
Исправив тип каталог на нужный мы не поменяли политику и всё работает в рамках уже установленной политики, что по моему мнению безопаснее.

Для исправления ошибки при создании ВМ было добавлено исправление типа контекста каталога /etc/named в плейбуке.

- name: Set selinux policy for directories '/etc/named'
    sefcontext:
      target: '/etc/named(/.*)?'
      setype: named_zone_t
      state: present

  - name: Apply new SELinux file context to filesystem
    command: restorecon -irv /etc/named