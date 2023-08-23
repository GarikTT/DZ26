<pre>
Урок 26 - Резервное копирование.
Цель занятия -
	обсудить политики и методики резерного копирования;
	работать с инструментами rsync, tar, dd и bacula.
Домашнее задание - настраиваем бэкапы.
Что нужно сделать?
	Настроить стенд Vagrant с двумя виртуальными машинами: backup_server и client.
	Настроить удаленный бекап каталога /etc c сервера client при помощи borgbackup. 
	Резервные копии должны соответствовать следующим критериям:
	директория для резервных копий /var/backup. Это должна быть отдельная точка монтирования. 
	В данном случае для демонстрации размер не принципиален, достаточно будет и 2GB;
	репозиторий дле резервных копий должен быть зашифрован ключом или паролем - на ваше усмотрение;
	имя бекапа должно содержать информацию о времени снятия бекапа;
	Глубина бекапа должна быть год, хранить можно по последней копии на конец месяца, кроме последних трех. 
	Последние три месяца должны содержать копии на каждый день. Т.е. должна быть правильно настроена политика удаления старых бэкапов;
	резервная копия снимается каждые 5 минут. Такой частый запуск в целях демонстрации;
	написан скрипт для снятия резервных копий. Скрипт запускается из соответствующей Cron джобы, либо systemd timer-а - на ваше усмотрение;
	настроено логирование процесса бекапа. Для упрощения можно весь вывод перенаправлять в logger с соответствующим тегом. 
	Если настроите не в syslog, то обязательна ротация логов.
	Запустите стенд на 30 минут.
	Убедитесь что резервные копии снимаются.
	Остановите бекап, удалите (или переместите) директорию /etc и восстановите ее из бекапа.
	Для сдачи домашнего задания ожидаем настроенные стенд, логи процесса бэкапа и описание процесса восстановления.
	Формат сдачи ДЗ - vagrant + ansible
Начинаем выполнять задание по шагам. Ага -
script --timing=time_loading_log loading.log
</pre>
1. Создаем файл ([Vagrantfile](Vagrantfile)) и запускаем его.
<pre>
vagrant up

В файле `/etc/ssh/sshd_config` на сервере "backup" и "client" установливаем следующие параметры:
PubkeyAuthentication yes
AuthorizedKeysFile  .ssh/authorized_keys
PasswordAuthentication yes
PermitEmptyPasswords yes
ChallengeResponseAuthentication no
Перезагружаем демона sshd -
systemctl restart sshd
2. Настраиваем сервер клиента для работы с сервером резервных копий.
vagrant ssh client
ssh borg@192.168.56.10
exit
# Эта команда осталась от попытки подключиться со стороны Linux Mint.
#ssh-keygen -f "/home/garik/.ssh/known_hosts" -R "192.168.56.10"
ssh-keygen
ssh-copy-id borg@192.168.56.10
ssh borg@192.168.56.10
exit
3. На всякий случай настроем аналогично сервер backup.
vagrant ssh backup
ssh vagrant@192.168.56.15
#ssh-keygen -f "/home/garik/.ssh/known_hosts" -R "192.168.56.15"
exit
ssh-keygen
ssh-copy-id vagrant@192.168.56.15
ssh vagrant@192.168.56.15
#ssh borg@192.168.56.10
exit
#chown borg:borg /var/backup/
exit
4. Настранваем службу borg на клиенте.
vagrant ssh client
Каталог /var/backup/ на сервере "backup" должен быть пустым!!! До этой ошибка я добирался целый день.
borg init --encryption=repokey borg@192.168.56.10:/var/backup/
borg create --stats --list borg@192.168.56.10:/var/backup/::etc-{now:%Y-%m-%d_%H:%M:%S} /etc
borg list borg@192.168.56.10:/var/backup/
borg extract borg@192.168.56.10:/var/backup/::etc-2023-08-23_01:02:19 etc/hostname // !!! Нужно скопировать имя архива!!!
5. Автоматизируем создание бэкапов с помощью systemd
Создаем сервис и таймер в каталоге /etc/systemd/system/

cat > /etc/systemd/system/borg-backup.service
[Unit]
Description=Borg Backup

[Service]
Type=oneshot

# Парольная фраза
Environment="BORG_PASSPHRASE=garik123"

# Репозиторий
Environment=REPO=borg@192.168.56.10:/var/backup/

# Что бэкапим
Environment=BACKUP_TARGET=/etc

# Создание бэкапа
ExecStart=/bin/borg create --stats ${REPO}::etc-{now:%%Y-%%m-%%d_%%H:%%M:%%S} ${BACKUP_TARGET}

# Проверка бэкапа
ExecStart=/bin/borg check ${REPO}

# Очистка старых бэкапов
#ExecStart=/bin/borg prune --keep-daily 90 --keep-monthly 12 --keep-yearly 1 ${REPO}
ExecStart=/bin/borg prune --keep-daily 90 --keep-monthly 3 --keep-yearly 1 ${REPO}

Глубина бекапа должна быть год, хранить можно по последней копии на конец месяца, кроме последних трех меяцев. 
Последние три месяца должны содержать копии на каждый день.

cat > /etc/systemd/system/borg-backup.timer (ссылка)
[Unit]
Description=Borg Backup

[Timer]
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target

Перезапускаем всех демонов -
systemctl daemon-reload

6. Включаем и запускаем службу таймера
systemctl enable borg-backup.timer 
systemctl start borg-backup.timer
systemctl status borg-backup.service
systemctl start borg-backup.service

7. Проверяем работу таймера
systemctl list-timers --all

8. cat > /home/vagrant/backupscript.sh
#!/bin/bash
BACKUPNAME="etc-$(date +'%Y-%m-%d%H:%M:%S')"
echo "Starting backup: $BACKUPNAME"
borg create --stats --list borg@192.168.56.10:/var/backup::$BACKUPNAME /etc
echo "Backup complete: $BACKUPNAME"

chmod +x /home/vagrant/backupscript.sh

9. Вставляем в файл /etc/crontab строку -
"*/2 * * * * root /home/vagrant/backupscript.sh >> /var/log/backup.log"
Честно говоря, ошибка была в обязательном символе перевода строки. И задача не хотела, блин,запускаться.

10. Настраиваем логирование процесса бекапа. Для этого в файле /etc/rsyslog.conf, находим строку #cron.* и заменяем ее на:
cron.*          /var/log/backup.log

После этого перезапускаем rsyslog командой:
systemctl restart rsyslog

11. Запустите стенд на 10 минут:
sleep 600
</pre>
12. Смотрим во все глаза на файл ([/var/log/backup.log](backup.log)) -
<pre>
cat /var/log/backup.log
Я в шоке, реально скрипт работает каждые 2 минуты!!!

14. Теперь самое сложное. Нужно удалить сохраняемый нами с вами каталог /etc на клиенте -
systemctl stop crond
systemctl status crond
rm -rf /etc
Затем пытаемся выполнить - 
borg list borg@192.168.56.10:/var/backup/
borg extract --list borg@192.168.56.10:/var/backup::etc-2023-08-22_00:06:05 /etc
systemctl start cron
Да, уж... Опять все сломалось. Теперь мы никогда не сможем теперь связаться с боргом!!!
Единственный вариант решения - установить заново "client" и уже потом, восстановить то, что мы сохранили!!!
15. Теперь создаем файл playbok.yml для первоначальной настройки серверов через ansible.
</pre>
Пишем файл inventory - ([hosts](hosts)).
<pre>
16. Все.
17. Теперь начинаем бороться с githab-ом для удобочитаемости README.md
</pre>
