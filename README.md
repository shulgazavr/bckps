# Настраиваем бэкапы

### Описание/Пошаговая инструкция выполнения домашнего задания:
Настроить стенд Vagrant с двумя виртуальными машинами: бэкап сервер - `bckp` и клиент - `clnt`.

Настроить удаленный бекап каталога `/etc` c сервера `clnt` при помощи `borgbackup`.

Резервные копии должны соответствовать следующим критериям:
- директория для резервных копий `/var/backup`. Это должна быть отдельная точка монтирования. В данном случае для демонстрации размер не принципиален, достаточно будет и 2GB;
- репозиторий дле резервных копий должен быть зашифрован ключом или паролем - на ваше усмотрение;
- имя бекапа должно содержать информацию о времени снятия бекапа;
- глубина бекапа должна быть год, хранить можно по последней копии на конец месяца, кроме последних трех. Последние три месяца должны содержать копии на каждый день, т.е. должна быть правильно настроена политика удаления старых бэкапов;
- резервная копия снимается каждые 5 минут. Такой частый запуск в целях демонстрации;
- написан скрипт для снятия резервных копий. Скрипт запускается из соответствующей Cron джобы, либо systemd timer-а - на ваше усмотрение;
- настроено логирование процесса бекапа. Для упрощения можно весь вывод перенаправлять в `logger` с соответствующим тегом. Если настроите не в `syslog`, то обязательна ротация логов.

1. Запустите стенд на 30 минут.
2. Убедитесь что резервные копии снимаются.
3. Остановите бекап, удалите (или переместите) директорию `/etc` и восстановите ее из бекапа.
4. Для сдачи домашнего задания ожидаем настроенные стенд, логи процесса бэкапа и описание процесса восстановления.
5. Формат сдачи ДЗ - vagrant + ansible


## Описание реализации.
[Vagrantfile](https://github.com/shulgazavr/bckps/blob/main/Vagrantfile), поднимающий 2 сервера: `clnt` и `bckp`. 
> Порядок имеет значение! Сначала `clnt`, потом `bckp`, т.к. только так средствами `ansible` удалось перенести публичный ключ на сервер.

Особенности н астройка параметров сети:
```
servers = [
    { :hostname => 'clnt',
      :ips => '192.168.31.202',
      :ram => 2048,
      :cpuz => 1
    },
    {   :hostname => 'bckp',
        :ips => '192.168.31.201',
        :ram => 2048,
        :cpuz => 2,
        :dfile => './sata1.vdi',
        :size => 2048
    }
]
...
            node.vm.network :public_network, ip: machine[:ips], bridge: "wlp58s0"
...
```
> Примечание: ip-адреса и имя интерфейса выбраны исходя из особенности окружения.

В зависимости от имени хоста, вызыввается нужный `playbook`: `playbook-clnt.yml`или `playbook-bckp.yml`)

После установки виртуальных машин, необходимо зайти на `clnt` и инициализировать репозиторий `borg`:

```
# borg init --encryption=repokey borg@192.168.31.201:/var/backup/
```
<details>
  <summary>Вывод команды</summary>

  ```
  The authenticity of host '192.168.31.201 (192.168.31.201)' can't be established.
ECDSA key fingerprint is SHA256:ziBP4DclaJB/fKLgKTOIpxQlwb/PQUDqorDONyqV9cs.
ECDSA key fingerprint is MD5:00:8a:bd:d3:ee:42:54:d0:37:5e:30:3b:bc:4a:bf:5a.
Are you sure you want to continue connecting (yes/no)? yes
Remote: Warning: Permanently added '192.168.31.201' (ECDSA) to the list of known hosts.
Enter new passphrase: 
Enter same passphrase again: 
Do you want your passphrase to be displayed for verification? [yN]: y
Your passphrase (between double-quotes): "123"
Make sure the passphrase displayed above is exactly what you wanted.

By default repositories initialized with this version will produce security
errors if written to with an older version (up to and including Borg 1.0.8).

If you want to use these older versions, you can disable the check by running:
borg upgrade --disable-tam ssh://borg@192.168.31.201/var/backup

See https://borgbackup.readthedocs.io/en/stable/changes.html#pre-1-0-9-manifest-spoofing-vulnerability for details about the security implications.

IMPORTANT: you will need both KEY AND PASSPHRASE to access this repo!
If you used a repokey mode, the key is stored in the repo, but you should back it up separately.
Use "borg key export" to export the key, optionally in printable format.
Write down the passphrase. Store both at safe place(s).
  ```
</details>

Создание нескольких бэкапов, проверка:
```
# borg create --stats --list borg@192.168.31.201:/var/backup/::"etc-{now:%Y-%m-%d_%H:%M:%S}" /etc
```
<details>
  <summary>Вывод команды</summary>

  ```
...
------------------------------------------------------------------------------
Archive name: etc-2023-03-01_20:07:50
Archive fingerprint: 0f6f6a7ec1f30482c680e26b9adefa5c08c4f8083785b71190fe0bef7e02e8f4
Time (start): Wed, 2023-03-01 20:07:53
Time (end):   Wed, 2023-03-01 20:07:56
Duration: 2.76 seconds
Number of files: 1713
Utilization of max. archive size: 0%
------------------------------------------------------------------------------
                       Original size      Compressed size    Deduplicated size
This archive:               28.51 MB             13.53 MB             11.88 MB
All archives:               28.51 MB             13.53 MB             11.88 MB

                       Unique chunks         Total chunks
Chunk index:                    1293                 1710
------------------------------------------------------------------------------
  ```
</details>

```
# borg list borg@192.168.31.201:/var/backup/
Enter passphrase for key ssh://borg@192.168.31.201/var/backup: 
etc-2023-03-01_20:07:50              Wed, 2023-03-01 20:07:53 [0f6f6a7ec1f30482c680e26b9adefa5c08c4f8083785b71190fe0bef7e02e8f4]
etc-2023-03-01_20:17:46              Wed, 2023-03-01 20:17:50 [af7afd59a22133810b624f25651456b6e7a05183593512b19b15e23d98fd384d]
```
Запуск сервиса, таймера и проверка:
```
# systemctl start borg-backup.service
```
```
# systemctl enable --now borg-backup.timer
```
```
# borg list borg@192.168.31.201:/var/backup/
Enter passphrase for key ssh://borg@192.168.31.201/var/backup: 
etc-2023-03-01_21:01:02              Wed, 2023-03-01 21:01:03 [69aabd7c1ecd6c6e39949820c02e3e1b99383c166c91a937db5ee406f2f6b58d]
```
Остановка бэкапа:
```
# systemctl stop borg-backup.timer
```
Удаление, восстановление, перемещение файла:
```
# rm -f /etc/hostname 
```
```
# borg extract borg@192.168.31.201:/var/backup/::etc-2023-03-01_21:03:01 etc/hostname
Enter passphrase for key ssh://borg@192.168.31.201/var/backup: 
```
```
# cp etc/hostname /etc/hostname
```
```
# ls -la /etc/hostname 
-rw-r--r--. 1 root root 5 Mar  1 21:11 /etc/hostname
```

> Примечание: для успешного извлечения файла, необходимо установить кодировку: `LANG=en_US.UTF-8`

Логирование процесса бэкапа осуществляется в `/var/log/messages`:
```
# tail -f /var/log/messages 
```
<details>
  <summary>Вывод команды</summary>

  ```
Mar  1 21:14:55 localhost systemd: Started Borg Backup.
Mar  1 21:16:00 localhost systemd: Starting Borg Backup...
Mar  1 21:16:03 localhost borg: ------------------------------------------------------------------------------
Mar  1 21:16:03 localhost borg: Archive name: etc-2023-03-01_21:16:01
Mar  1 21:16:03 localhost borg: Archive fingerprint: feea6d565911cdd386af2a7c643723eb2ce179d00856659c8158254740c64cf0
Mar  1 21:16:03 localhost borg: Time (start): Wed, 2023-03-01 21:16:02
Mar  1 21:16:03 localhost borg: Time (end):   Wed, 2023-03-01 21:16:03
Mar  1 21:16:03 localhost borg: Duration: 0.56 seconds
Mar  1 21:16:03 localhost borg: Number of files: 1714
Mar  1 21:16:03 localhost borg: Utilization of max. archive size: 0%
Mar  1 21:16:03 localhost borg: ------------------------------------------------------------------------------
Mar  1 21:16:03 localhost borg: Original size      Compressed size    Deduplicated size
Mar  1 21:16:03 localhost borg: This archive:               28.52 MB             13.53 MB             64.31 kB
Mar  1 21:16:03 localhost borg: All archives:               57.03 MB             27.07 MB             11.94 MB
Mar  1 21:16:03 localhost borg: Unique chunks         Total chunks
Mar  1 21:16:03 localhost borg: Chunk index:                    1294                 3420
Mar  1 21:16:03 localhost borg: ------------------------------------------------------------------------------
  ```
</details>


