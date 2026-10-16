# OTUS Linux Professional — Резервное копирование (BorgBackup)

## Задание

1. Настроить стенд Vagrant с двумя ВМ: `backup_server` и `client`.
2. Настроить удалённый бэкап каталога `/etc` с `client` при помощи borgbackup:
   - директория для бэкапов `/var/backup` — отдельная точка монтирования (2 ГБ);
   - репозиторий зашифрован ключом или паролем;
   - имя бэкапа содержит время снятия;
   - глубина — год: последние 3 месяца по копии на каждый день, далее по последней копии на конец месяца;
   - бэкап каждые 5 минут;
   - скрипт бэкапа запускается из cron или systemd timer;
   - логирование процесса бэкапа (logger с тегом).
3. Запустить стенд на 30 минут, убедиться, что бэкапы снимаются.
4. Остановить бэкап, удалить `/etc` и восстановить его из бэкапа.

Формат сдачи — Vagrant + Ansible.

## Схема сети

~~~
                Хост otus-1 (Ubuntu 26.04, VirtualBox 7.2)
                                 |
                 host-only сеть 192.168.11.0/24
                 |                              |
     +-----------------------+      +-----------------------------+
     | client                |      | backup_server               |
     | 192.168.11.150        | ssh  | 192.168.11.160              |
     | borg client           |----->| borg serve (user borg)      |
     | /etc -> бэкап         |      | /var/backup = /dev/sdc 2 ГБ |
     +-----------------------+      +-----------------------------+
~~~

## Структура репозитория

| Путь | Назначение |
|------|------------|
| [Vagrantfile](Vagrantfile) | Стенд: 2 ВМ Ubuntu 22.04 + диск 2 ГБ для backup_server |
| [ansible/](ansible/) | Playbook настройки обеих ВМ |
| [ansible/group_vars/](ansible/group_vars/) | Переменные (IP, репозиторий, пароль, политика хранения) |
| [ansible/templates/](ansible/templates/) | Скрипт бэкапа, systemd service и timer |
| [logs/](logs/) | Логи процесса бэкапа и восстановления |
| [screenshots/](screenshots/) | Скриншоты выполнения |

## Особенности реализации

- **Отдельная точка монтирования.** Vagrant создаёт диск 2 ГБ, Ansible находит его по размеру, форматирует в ext4 (метка `BACKUP`) и монтирует в `/var/backup` через `/etc/fstab`. `lost+found` удаляется, так как `borg init` требует пустой каталог.
- **Доступ.** На client генерируется ключ ed25519. На backup_server ключ добавлен пользователю `borg` с ограничением `command="borg serve --restrict-to-path /var/backup",restrict`: shell по этому ключу недоступен.
- **Шифрование.** `borg init --encryption=repokey`, пароль `Otus1234`. Копия ключа экспортирована в `/root/borg-repokey.txt` на client.
- **Имя архива:** `etc-{now:%Y-%m-%d_%H:%M:%S}`.
- **Политика хранения:** `--keep-daily 90 --keep-monthly 9`. Borg не засчитывает архивы, оставленные правилом daily, в правило monthly. Итого 3 месяца ежедневных копий + 9 месячных = 1 год. После `prune` выполняется `borg compact`, затем `borg check`.
- **Расписание:** systemd timer `borg-backup.timer` (`OnCalendar=*:0/5`) запускает `borg-backup.service`, который выполняет `/usr/local/bin/borg-backup.sh`.
- **Логирование:** весь вывод скрипта уходит в `logger -t borg-backup` (syslog/journald), ротацию выполняет система. Просмотр: `journalctl -t borg-backup` или `grep borg-backup /var/log/syslog`.

## Запуск

~~~bash
vagrant up
vagrant ssh client -c 'sudo systemctl list-timers borg-backup.timer'
~~~

## Восстановление /etc

Описание процесса восстановления — в разделе ниже (заполняется по результатам этапа восстановления).
