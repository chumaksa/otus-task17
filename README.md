# otus-task17

# Настраиваем бэкапы

Настроить стенд Vagrant с двумя виртуальными машинами: backup_server и client.

### Подготовка.

Для выполнения задания будем использовать виртуальные машины развёрнутые с помощью vagrant. \
Vagrantfile имеет следующий вид:
```

# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"
  
  config.vm.define "backup" do |backup|
    backup.vm.hostname = "backup"
    backup.vm.network "private_network", ip: "192.168.56.160"
    config.vm.disk :disk, size: "2GB", name: "backup_storage"
  end
  
  config.vm.define "client" do |client|
    client.vm.hostname = "client"
    client.vm.network "private_network", ip: "192.168.56.150"
  end
  
end
```

Для предварительной подготовки машин будем использовать Ansible. \
Все необхдимые файлы есть в репозитории. \
После равёртования и предварительной настройки тестового стенда перейдём непосредственно к выполнению задания.

### Решение.

Подключение к backup с client будем выполнять с помощью ssh-ключей. \
Для этого необходимо подключиться к client и сгенерировать на нём SSH-ключи.
```

[vagrant@client ~]$ su -l
Password: 
Last login: Mon Mar  4 14:26:44 UTC 2024 on pts/0
[root@client ~]# ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:T8WXIj0TRlDRsVr2obURVlSLWBVu2IxTOKXbmC36oQc root@client
The key's randomart image is:
+---[RSA 2048]----+
|          .+*+=XB|
|           +o+%oo|
|          ..B*BX |
|           o O@.+|
|        S . .* +.|
|         o E. .  |
|          ....   |
|            o..  |
|           ...   |
+----[SHA256]-----+

[root@client ~]# cat /root/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDtS9ZrshB3GtHDbdkUkUm5a+sH/SfU/6DUsq5ZFl0iUfJLhoQ/aL+fbY7Pv2cHsdKSDLnld57o3ftD3zHA7Sn/HjcDhYIGuYEGtr9Hsuee7rlK7vRrW4kTHSwjR1wS1UKFKBVjPwqqXCXVct1xOSNeBskqU2qF2AO7HfjGDQGUfM5BdgK5xyr/5r0KruEAcOkEdfYt1v1nnbQ6L63wl+nN2saRcdN4slUpofDmtOwvHYH9CapbMahyOg7BqvdZpwKFPXM/b7JFwEpBlQB1L5y8HE7sFRjADw+pC/xVUzAQQJxzbxuTNtS0kkQjEh5SzEY2cYvHdMSnePOKlzQiN5sj root@client
```

Далее на сервере бэкапов нам необходимо залогиниться под пользователем borg и скопировать ключ с client.
```

[vagrant@backup ~]$ su -l
Password: 
[root@backup ~]# su -l borg
[borg@backup ~]$ mkdir .ssh
[borg@backup ~]$ touch .ssh/authorized_keys
[borg@backup ~]$ chmod 700 .ssh
[borg@backup ~]$ chmod 600 .ssh/authorized_keys

[borg@backup ~]$ vi .ssh/authorized_keys
[borg@backup ~]$ cat .ssh/authorized_keys
command="/usr/bin/borg serve" ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDtS9ZrshB3GtHDbdkUkUm5a+sH/SfU/6DUsq5ZFl0iUfJLhoQ/aL+fbY7Pv2cHsdKSDLnld57o3ftD3zHA7Sn/HjcDhYIGuYEGtr9Hsuee7rlK7vRrW4kTHSwjR1wS1UKFKBVjPwqqXCXVct1xOSNeBskqU2qF2AO7HfjGDQGUfM5BdgK5xyr/5r0KruEAcOkEdfYt1v1nnbQ6L63wl+nN2saRcdN4slUpofDmtOwvHYH9CapbMahyOg7BqvdZpwKFPXM/b7JFwEpBlQB1L5y8HE7sFRjADw+pC/xVUzAQQJxzbxuTNtS0kkQjEh5SzEY2cYvHdMSnePOKlzQiN5sj root@client
```

Далее переходим на client и выполняем инициализацию репозитория borg.\
В качестве пароля укажем - Otus1234.
```

[root@client ~]# borg init --encryption=repokey borg@192.168.56.160:/var/backup/
Enter new passphrase: 
Enter same passphrase again: 
Do you want your passphrase to be displayed for verification? [yN]: y
Your passphrase (between double-quotes): "Otus1234"
Make sure the passphrase displayed above is exactly what you wanted.

By default repositories initialized with this version will produce security
errors if written to with an older version (up to and including Borg 1.0.8).

If you want to use these older versions, you can disable the check by running:
borg upgrade --disable-tam ssh://borg@192.168.56.160/var/backup

See https://borgbackup.readthedocs.io/en/stable/changes.html#pre-1-0-9-manifest-spoofing-vulnerability for details about the security implications.

IMPORTANT: you will need both KEY AND PASSPHRASE to access this repo!
If you used a repokey mode, the key is stored in the repo, but you should back it up separately.
Use "borg key export" to export the key, optionally in printable format.
Write down the passphrase. Store both at safe place(s).
```

Запустит тестовый бэкап и проверим работоспособность.
```

[root@client ~]# borg create --stats --list borg@192.168.56.160:/var/backup/::"etc-{now:%Y-%m-%d_%H:%M:%S}" /etc
Enter passphrase for key ssh://borg@192.168.56.160/var/backup: 
A /etc/crypttab
s /etc/mtab
A /etc/hostname
***
***
***
A /etc/sudoers.d/vagrant
d /etc/sudoers.d
d /etc
------------------------------------------------------------------------------
Archive name: etc-2024-03-04_14:42:42
Archive fingerprint: 41d8ea4a9b2f6c32bea7233cf36d4004667c2c815104f9c6f4311585f9257be6
Time (start): Mon, 2024-03-04 14:42:53
Time (end):   Mon, 2024-03-04 14:42:55
Duration: 2.54 seconds
Number of files: 1698
Utilization of max. archive size: 0%
------------------------------------------------------------------------------
                       Original size      Compressed size    Deduplicated size
This archive:               28.43 MB             13.49 MB             11.84 MB
All archives:               28.43 MB             13.49 MB             11.84 MB

                       Unique chunks         Total chunks
Chunk index:                    1280                 1698
------------------------------------------------------------------------------

[root@client ~]# borg list borg@192.168.56.160:/var/backup/
Enter passphrase for key ssh://borg@192.168.56.160/var/backup: 
etc-2024-03-04_14:42:42              Mon, 2024-03-04 14:42:53 [41d8ea4a9b2f6c32bea7233cf36d4004667c2c815104f9c6f4311585f9257be6]
```

Далее выполним автоматизацию создания бэкапов с помощью systemd. \
Для этого создадим на client сервис и таймер.
```

[root@client ~]# vi /etc/systemd/system/borg-backup.service
[root@client ~]# cat /etc/systemd/system/borg-backup.service
[Unit]
Description=Borg Backup
Wants=borg-backup.timer

[Service]
Type=oneshot

# Парольная фраза
Environment="BORG_PASSPHRASE=Otus1234"
# Репозиторий
Environment=REPO=borg@192.168.56.160:/var/backup/
# Что бэкапим
Environment=BACKUP_TARGET=/etc

# Создание бэкапа
ExecStart=/bin/borg create \
    --stats                \
    ${REPO}::etc-{now:%%Y-%%m-%%d_%%H:%%M:%%S} ${BACKUP_TARGET}

# Проверка бэкапа
ExecStart=/bin/borg check ${REPO}

# Очистка старых бэкапов
ExecStart=/bin/borg prune \
    --keep-daily  90      \
    --keep-monthly 12     \
    --keep-yearly  1       \
    ${REPO}

[Install]
WantedBy=multi-user.target
    

[root@client ~]# cat /etc/systemd/system/borg-backup.timer 
[Unit]
Description=Borg Backup
Requires=borg-backup.service

[Timer]
OnUnitActiveSec=5min
Unit=borg-backup.service

[Install]
WantedBy=timers.target
```

Выполним запуск и после некоторого времени проверим наличие бэкапов.
```

[root@client ~]# systemctl enable borg-backup.timer --now
Created symlink from /etc/systemd/system/timers.target.wants/borg-backup.timer to /etc/systemd/system/borg-backup.timer.
[root@client ~]# systemctl status borg-backup.timer
● borg-backup.timer - Borg Backup
   Loaded: loaded (/etc/systemd/system/borg-backup.timer; enabled; vendor preset: disabled)
   Active: active (waiting) since Mon 2024-03-04 15:09:26 UTC; 22s ago

Mar 04 15:09:26 client systemd[1]: Started Borg Backup.

[root@client ~]# borg list borg@192.168.56.160:/var/backup/
Enter passphrase for key ssh://borg@192.168.56.160/var/backup: 
etc-2024-03-04_15:14:54              Mon, 2024-03-04 15:14:55 [9d78cd2a10109046cc2fc004d481a684926140b641193406ca6b8d4736b76651]
etc-2024-03-04_15:19:56              Mon, 2024-03-04 15:19:57 [a1e076241fd5d8936cccef2cc1229368404d2196f38ab7036536676279ab4f63]
```

Выполнить воостановление файла, например etc/hostname можно следующей командой.
```

[root@client ~]# borg extract borg@192.168.56.160:/var/backup/::etc-2024-03-04_15:37:54 etc/hostname
Enter passphrase for key ssh://borg@192.168.56.160/var/backup: 
[root@client ~]# 
```



