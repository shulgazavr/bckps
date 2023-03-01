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
