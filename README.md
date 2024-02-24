# otus-task15

# PAM

1. Запретить всем пользователям, кроме группы admin, логин в выходные (суббота и воскресенье), без учета праздников.

Для выполнения задания будем использовать VM с ОС Centos 7. \
Развернём тестовый стенд с помощью Vagrant. Для этого создадим Vagranfile следующего содержания:
```

# Описание параметров ВМ
MACHINES = {
  # Имя DV "pam"
  :"pam" => {
              # VM box
              :box_name => "centos/7",
              #box_version
              #:box_version => "20210210.0",
              # Количество ядер CPU
              :cpus => 2,
              # Указываем количество ОЗУ (В Мегабайтах)
              :memory => 1024,
              # Указываем IP-адрес для ВМ
              :ip => "192.168.57.10",
            }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    # Отключаем сетевую папку
    config.vm.synced_folder ".", "/vagrant", disabled: true
    # Добавляем сетевой интерфейс
    config.vm.network "private_network", ip: boxconfig[:ip]
    # Применяем параметры, указанные выше
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.box_version = boxconfig[:box_version]
      box.vm.host_name = boxname.to_s

      box.vm.provider "virtualbox" do |v|
        v.memory = boxconfig[:memory]
        v.cpus = boxconfig[:cpus]
      end
      box.vm.provision "shell", inline: <<-SHELL
          #Разрешаем подключение пользователей по SSH с использованием пароля
          sed -i 's/^PasswordAuthentication.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config
          #Перезапуск службы SSHD
          systemctl restart sshd.service
  	  SHELL
    end
  end
end
```

## Решение

В нашей VM создадим двух пользователей (otus - пользователь, otusadm - администратор) и зададим им пароли.
```

[root@pam ~]# useradd otus
[root@pam ~]# passwd otus
Changing password for user otus.
New password: 
BAD PASSWORD: The password contains the user name in some form
Retype new password: 
passwd: all authentication tokens updated successfully.

[root@pam ~]# useradd otusadm
Creating mailbox file: File exists
[root@pam ~]# passwd otusadm 
Changing password for user otusadm.
New password: 
Retype new password: 
passwd: all authentication tokens updated successfully.
```

Далее создадим группу admin и добавим в неё пользователей: vagrant, root, otusadm.
```

[root@pam ~]# groupadd admin
[root@pam ~]# usermod vagrant -aG admin 
[root@pam ~]# usermod root -aG admin 
[root@pam ~]# usermod otusadm -aG admin 
```

Пока мы не настраивали ограничений и оба пользователя должны подключаться по ssh к нашей VM.\
Проверим это путём подключения с хостовой машины.
```

chumaksa@debpc:~/otus/otus-task15$ ssh otus@192.168.57.10
otus@192.168.57.10's password: 
Last failed login: Sat Feb 24 12:48:54 UTC 2024 on pts/0
There were 5 failed login attempts since the last successful login.
[otus@pam ~]$ exit
logout
Connection to 192.168.57.10 closed.

chumaksa@debpc:~/otus/otus-task15$ ssh otusadm@192.168.57.10
otusadm@192.168.57.10's password: 
Last failed login: Sat Feb 24 12:42:31 UTC 2024 from 192.168.57.1 on ssh:notty
There was 1 failed login attempt since the last successful login.
[otusadm@pam ~]$ exit
logout
Connection to 192.168.57.10 closed.
```

Далее настроим правило, по которому все пользователи кроме тех, что указаны в группе admin не смогут подключаться в выходные дни. \
Проверим, что наши пользователи vagrant, root, otusadm находятся в группе admin.
```

[root@pam ~]# cat /etc/group | grep admin
printadmin:x:997:
admin:x:1003:vagrant,root,otusadm
```

Для ограничения аутентификации будем использовать pam_exec вместе с небольшим скриптом. \
Создадим файл-скрипт /usr/local/bin/login.sh и добавим ему права на исполнение.
```

[root@pam ~]# vi /usr/local/bin/login.sh
[root@pam ~]# cat /usr/local/bin/login.sh
#!/bin/bash
#Первое условие: если день недели суббота или воскресенье
if [ $(date +%a) = "Sat" ] || [ $(date +%a) = "Sun" ]; then
 #Второе условие: входит ли пользователь в группу admin
 if getent group admin | grep -qw "$PAM_USER"; then
        #Если пользователь входит в группу admin, то он может подключиться
        exit 0
      else
        #Иначе ошибка (не сможет подключиться)
        exit 1
    fi
  #Если день не выходной, то подключиться может любой пользователь
  else
    exit 0
fi

[root@pam ~]# chmod +x /usr/local/bin/login.sh
[root@pam ~]# ls -l /usr/local/bin/login.sh
-rwxr-xr-x. 1 root root 719 Feb 24 13:40 /usr/local/bin/login.sh
```

Добавим в файл /etc/pam.d/sshd модуль pam_exec и наше скрипт.
```

[root@pam ~]# vi /etc/pam.d/sshd 
[root@pam ~]# cat /etc/pam.d/sshd
#%PAM-1.0
auth	   required	pam_sepermit.so
auth       substack     password-auth
auth       include      postlogin
# Used with polkit to reauthorize users in remote sessions
-auth      optional     pam_reauthorize.so prepare
account	   required     pam_exec.so /usr/local/bin/login.sh	
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      password-auth
session    include      postlogin
# Used with polkit to reauthorize users in remote sessions
-session   optional     pam_reauthorize.so prepare
```

Всё готово. Выполним проверку, попробуем подключится к VM с хоста.\
Проверка проводилас в выходной день.
```

chumaksa@debpc:~/otus/otus-task15$ ssh otusadm@192.168.57.10
otusadm@192.168.57.10's password: 
Last login: Sat Feb 24 13:10:20 2024 from 192.168.57.1
[otusadm@pam ~]$ exit
logout
Connection to 192.168.57.10 closed.

chumaksa@debpc:~/otus/otus-task15$ ssh otus@192.168.57.10
otus@192.168.57.10's password: 
/usr/local/bin/login.sh failed: exit code 1
Connection closed by 192.168.57.10 port 22
```

Дополнительно проверим журнал и убедимся, что наше ограничение работает корректно.
```

[root@pam ~]# journalctl -f
-- Logs begin at Sat 2024-02-24 12:16:27 UTC. --
Feb 24 13:55:36 pam systemd-logind[399]: New session 12 of user otusadm.
Feb 24 13:55:36 pam sshd[22464]: pam_unix(sshd:session): session opened for user otusadm by (uid=0)
Feb 24 13:55:41 pam sshd[22472]: Received disconnect from 192.168.57.1 port 34288:11: disconnected by user
Feb 24 13:55:41 pam sshd[22472]: Disconnected from 192.168.57.1 port 34288
Feb 24 13:55:41 pam sshd[22464]: pam_unix(sshd:session): session closed for user otusadm
Feb 24 13:55:41 pam systemd-logind[399]: Removed session 12.
Feb 24 13:55:41 pam systemd[1]: Removed slice User Slice of otusadm.
Feb 24 13:55:55 pam sshd[22497]: pam_exec(sshd:account): /usr/local/bin/login.sh failed: exit code 1
Feb 24 13:55:55 pam sshd[22497]: Failed password for otus from 192.168.57.1 port 48632 ssh2
Feb 24 13:55:55 pam sshd[22497]: fatal: Access denied for user otus by PAM account configuration [preauth]
```





