Создаем виртуальную машину с помощью VagrantFile, запускаем, заходим

vagrant arjundandagi/centos-7-nginx
vagrant up
vagrant ssh

Заходим под рутом, бновлеям VM, устанавливаем необходимое ПО, проверяем статус службы nginx, при необходимости запускаем, проверяем статус SELinux

sudo su
yum update

Установка инструментов для SELinux
yum install -y setroubleshoot-server selinux-policy-mls setools-console policycoreutils-newrole
yum install setools-console


systemctl status nginx.service
systemctl start nginx.service
sestatus

Настроем nginx на прослушивание порт 90. Перезапустив службу выдаст ошибку:

[root@localhost vagrant]# systemctl restart nginx.service 
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.

Запустим журнал
[root@localhost vagrant]# journalctl -xe

В журнале сразу написано, что port 90 не разрешен и как решить данную проблему

Nov 24 11:53:49 localhost.localdomain dbus[363]: [system] Successfully activated service 'org.fedoraproject.Setroubleshootd'
Nov 24 11:53:49 localhost.localdomain setroubleshoot[8941]: SELinux is preventing /usr/sbin/nginx from name_bind access on the tcp_socket port 90. For complete SELinux messages run: sealert -l 5bde7882-dd21-4c72-b736-7cb1a4611754
Nov 24 11:53:49 localhost.localdomain python[8941]: SELinux is preventing /usr/sbin/nginx from name_bind access on the tcp_socket port 90.
                                                    
                                                    *****  Plugin bind_ports (99.5 confidence) suggests   ************************
                                                    
                                                    If you want to allow /usr/sbin/nginx to bind to network port 90
                                                    Then you need to modify the port type.
                                                    Do
                                                    # semanage port -a -t PORT_TYPE -p tcp 90
                                                        where PORT_TYPE is one of the following: http_cache_port_t, http_port_t, jboss_management_port_t, jboss_messaging_port_t, ntop_port_t, puppet_port_t.
                                                    
                                                    *****  Plugin catchall (1.49 confidence) suggests   **************************
                                                    
                                                    If you believe that nginx should be allowed name_bind access on the port 90 tcp_socket by default.
                                                    Then you should report this as a bug.
                                                    You can generate a local policy module to allow this access.
                                                    Do
                                                    allow this access for now by executing:
                                                    # ausearch -c 'nginx' --raw | audit2allow -M my-nginx
                                                    # semodule -i my-nginx.pp
                                                    

Для просмотра полной информации по ошибки можно использовать предложенную команду

[root@localhost vagrant]# sealert -l 5bde7882-dd21-4c72-b736-7cb1a4611754

Вывод:
SELinux is preventing /usr/sbin/nginx from name_bind access on the tcp_socket port 90.

*****  Plugin bind_ports (99.5 confidence) suggests   ************************

If you want to allow /usr/sbin/nginx to bind to network port 90
Then you need to modify the port type.
Do
# semanage port -a -t PORT_TYPE -p tcp 90
    where PORT_TYPE is one of the following: http_cache_port_t, http_port_t, jboss_management_port_t, jboss_messaging_port_t, ntop_port_t, puppet_port_t.

*****  Plugin catchall (1.49 confidence) suggests   **************************

If you believe that nginx should be allowed name_bind access on the port 90 tcp_socket by default.
Then you should report this as a bug.
You can generate a local policy module to allow this access.
Do
allow this access for now by executing:
# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
# semodule -i my-nginx.pp


Additional Information:
Source Context                system_u:system_r:httpd_t:s0
Target Context                system_u:object_r:reserved_port_t:s0
Target Objects                port 90 [ tcp_socket ]
Source                        nginx
Source Path                   /usr/sbin/nginx
Port                          90
Host                          localhost.localdomain
Source RPM Packages           nginx-1.20.1-10.el7.x86_64
Target RPM Packages           
Policy RPM                    selinux-policy-3.13.1-268.el7_9.2.noarch
Selinux Enabled               True
Policy Type                   targeted
Enforcing Mode                Enforcing
Host Name                     localhost.localdomain
Platform                      Linux localhost.localdomain
                              3.10.0-1160.119.1.el7.x86_64 #1 SMP Tue Jun 4
                              14:43:51 UTC 2024 x86_64 x86_64
Alert Count                   1
First Seen                    2024-11-24 11:53:49 UTC
Last Seen                     2024-11-24 11:53:49 UTC
Local ID                      5bde7882-dd21-4c72-b736-7cb1a4611754

Raw Audit Messages
type=AVC msg=audit(1732449229.446:684): avc:  denied  { name_bind } for  pid=8935 comm="nginx" src=90 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:reserved_port_t:s0 tclass=tcp_socket permissive=0


type=SYSCALL msg=audit(1732449229.446:684): arch=x86_64 syscall=bind success=no exit=EACCES a0=6 a1=55a74bcccb60 a2=10 a3=7fff4f5ab960 items=0 ppid=1 pid=8935 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm=nginx exe=/usr/sbin/nginx subj=system_u:system_r:httpd_t:s0 key=(null)

Hash: nginx,httpd_t,reserved_port_t,tcp_socket,name_bind

Другой способ посмотреть причину поломки сервиса:

[root@localhost vagrant]# audit2why < /var/log/audit/audit.log

Вывод:

type=AVC msg=audit(1732449229.446:684): avc:  denied  { name_bind } for  pid=8935 comm="nginx" src=90 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:reserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

РЕШНИЕ:
Способ 1 - Создаем собственный модуль my-nginx и применяем его (политика общего характера, не точечная, будет разрешать всем портам в nginx работать)

ausearch -c 'nginx' --raw | audit2allow -M my-nginx
semodule -i my-nginx.pp

Результат:
[root@localhost vagrant]# systemctl restart nginx.service 
[root@localhost vagrant]# ss -ntlpZ
State      Recv-Q Send-Q                                                                         Local Address:Port                                                                                        Peer Address:Port              
LISTEN     0      128                                                                                        *:111                                                                                                    *:*                   users:(("rpcbind",pid=379,proc_ctx=system_u:system_r:rpcbind_t:s0,fd=8))
LISTEN     0      128                                                                                        *:22                                                                                                     *:*                   users:(("sshd",pid=655,proc_ctx=system_u:system_r:sshd_t:s0-s0:c0.c1023,fd=3))
LISTEN     0      100                                                                                127.0.0.1:25                                                                                                     *:*                   users:(("master",pid=836,proc_ctx=system_u:system_r:postfix_master_t:s0,fd=13))
LISTEN     0      128                                                                                        *:90                                                                                                     *:*                   users:(("nginx",pid=9022,proc_ctx=system_u:system_r:httpd_t:s0,fd=6),("nginx",pid=9020,proc_ctx=system_u:system_r:httpd_t:s0,fd=6))
LISTEN     0      128                                                                                     [::]:111                                                                                                 [::]:*                   users:(("rpcbind",pid=379,proc_ctx=system_u:system_r:rpcbind_t:s0,fd=11))
LISTEN     0      128                                                                                     [::]:22                                                                                                  [::]:*                   users:(("sshd",pid=655,proc_ctx=system_u:system_r:sshd_t:s0-s0:c0.c1023,fd=4))
LISTEN     0      100                                                                                    [::1]:25                                                                                                  [::]:*                   users:(("master",pid=836,proc_ctx=system_u:system_r:postfix_master_t:s0,fd=14))
LISTEN     0      128                                                                                     [::]:90                                                                                                  [::]:*                   users:(("nginx",pid=9022,proc_ctx=system_u:system_r:httpd_t:s0,fd=7),("nginx",pid=9020,proc_ctx=system_u:system_r:httpd_t:s0,fd=7))
[root@localhost vagrant]# semodule -l | grep nginx
my-nginx        1.0

Удалить модуль:

semodule -r my-nginx

Способ 2 Добавить порт 90 в разрешенные (Более точечный и рекомендованный сособ. Добавляет определнный порт в метку типа http_port_t)

semanage port -a -t http_port_t -p tcp 90

[root@localhost vagrant]# seinfo --portcon=90
        portcon tcp 90 system_u:object_r:http_port_t:s0
        portcon tcp 1-511 system_u:object_r:reserved_port_t:s0
        portcon udp 1-511 system_u:object_r:reserved_port_t:s0
        portcon sctp 1-511 system_u:object_r:reserved_port_t:s0

Удалить порт можно:

semanage port -d -t ssh_port_t -p tcp 90

Способ 3: Параметризованные политики

Посмотрим ЛОГ

[root@localhost vagrant]# audit2why < /var/log/audit/audit.log | grep 90
    type=AVC msg=audit(1732449229.446:684): avc:  denied  { name_bind } for  pid=8935 comm="nginx" src=90 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:reserved_port_t:s0 tclass=tcp_socket permissive=0
    type=AVC msg=audit(1732451941.172:699): avc:  denied  { name_bind } for  pid=9043 comm="nginx" src=90 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:reserved_port_t:s0 tclass=tcp_socket permissive=0
    type=AVC msg=audit(1732453475.932:718): avc:  denied  { name_bind } for  pid=9155 comm="nginx" src=90 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:reserved_port_t:s0 tclass=tcp_socket permissive=0
    type=AVC msg=audit(1732453553.407:724): avc:  denied  { name_bind } for  pid=9201 comm="nginx" src=90 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:reserved_port_t:s0 tclass=tcp_socket permissive=0
    type=AVC msg=audit(1732454009.720:726): avc:  denied  { name_bind } for  pid=9230 comm="nginx" src=90 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:reserved_port_t:s0 tclass=tcp_socket permissive=0
[root@localhost vagrant]# grep 1732449229.446:684 /var/log/audit/audit.log | audit2why
    type=AVC msg=audit(1732449229.446:684): avc:  denied  { name_bind } for  pid=8935 comm="nginx" src=90 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:reserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

[root@localhost vagrant]# grep 1732451941.172:699 /var/log/audit/audit.log | audit2why
    type=AVC msg=audit(1732451941.172:699): avc:  denied  { name_bind } for  pid=9043 comm="nginx" src=90 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:reserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

    В методичке указано, что в резултате фильра должно быть указано значение nis_enable, которе нужно сменить, чтобы порт заработал. В нашем фильтре это значение не указано. 
    Предлагается создать и добавить модуль  в разрешенные. Попробуем nis_enable найти и включить.

[root@localhost vagrant]# semanage boolean -l | grep nis
    varnishd_connect_any           (off  ,  off)  Allow varnishd to connect any
    nis_enabled                    (off  ,  off)  Allow nis to enabled
[root@localhost vagrant]# setsebool -P nis_enabled on
[root@localhost vagrant]# systemctl restart nginx.service 
    Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.


В данном случаем мы не смогли найти необходимый параметр, который мог бы помочь включить разрешение.