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

```
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
```

## Структура репозитория

| Путь | Назначение |
|------|------------|
| [Vagrantfile](Vagrantfile) | Стенд: 2 ВМ Ubuntu 22.04 + диск 2 ГБ для backup_server |
| [ansible/](ansible/) | Playbook настройки обеих ВМ |
| [ansible/group_vars/](ansible/group_vars/) | Переменные (IP, репозиторий, пароль, политика хранения) |
| [ansible/templates/](ansible/templates/) | Скрипт бэкапа, systemd service и timer |
| [logs/](logs/) | Логи развёртывания, бэкапа и восстановления |
| [screenshots/](screenshots/) | Скриншоты выполнения |

## Особенности реализации

- **Отдельная точка монтирования.** Vagrant создаёт диск 2 ГБ, Ansible находит его по размеру, форматирует в ext4 (метка `BACKUP`) и монтирует в `/var/backup` через `/etc/fstab`. `lost+found` удаляется, так как `borg init` требует пустой каталог.
- **Доступ.** На client генерируется ключ ed25519. На backup_server он добавлен пользователю `borg` с ограничением `command="borg serve --restrict-to-path /var/backup",restrict`: shell по этому ключу недоступен.
- **Шифрование.** `borg init --encryption=repokey`, пароль `Otus1234`. Копия ключа экспортирована в `/root/borg-repokey.txt` на client.
- **Имя архива:** `etc-{now:%Y-%m-%d_%H:%M:%S}`, например `etc-2026-10-16_13:35:00`.
- **Политика хранения:** `--keep-daily 90 --keep-monthly 9`. Borg не засчитывает архивы, оставленные правилом daily, в правило monthly. Итого 3 месяца ежедневных копий + 9 месячных = 1 год. После `prune` выполняются `borg compact` и `borg check`.
- **Расписание:** systemd timer [borg-backup.timer](ansible/templates/borg-backup.timer.j2) (`OnCalendar=*:0/5`) запускает [borg-backup.service](ansible/templates/borg-backup.service.j2), который выполняет скрипт [borg-backup.sh](ansible/templates/borg-backup.sh.j2).
- **Логирование:** весь вывод скрипта уходит в `logger -t borg-backup` (syslog/journald), ротацию выполняет система. Скрипт перед выходом дожидается завершения `logger`: иначе systemd завершает его вместе с сервисом и последние строки лога теряются.

## Запуск

```bash
vagrant up
```

## Результаты

- За 30+ минут работы стенда выполнено 11 бэкапов каждые 5 минут, все с `exit code: 0`: [logs/borg-backup.log](logs/borg-backup.log), [logs/borg-backup-syslog.log](logs/borg-backup-syslog.log).
- В репозитории остаётся 2 архива: самый старый (`daily[oldest]`) и последний за текущий день (`daily #1`). Промежуточные архивы за тот же день удаляет `prune`, как и требует политика «одна копия на день».
- Лог развёртывания: [logs/vagrant-up.log](logs/vagrant-up.log), [logs/vagrant-provision-fix.log](logs/vagrant-provision-fix.log).

## Восстановление /etc

Все команды выполняются на client от root в одной сессии.

```bash
# 1. Остановить бэкап
systemctl stop borg-backup.timer

# 2. Подключение к репозиторию, последний архив, контрольные суммы /etc
export BORG_REPO='borg@192.168.11.160:/var/backup/' BORG_PASSPHRASE=Otus1234 \
       BORG_RSH='ssh -i /root/.ssh/id_ed25519 -o BatchMode=yes'
ARCHIVE=$(borg list --short --last 1)
(cd / && find etc -type f -print0 | sort -z | xargs -0 md5sum) > /root/etc-before.md5

# 3. «Удалить» /etc
mv /etc /etc.old
borg list    # -> Remote: No user exists for uid 0

# 4. Восстановить. Без /etc/passwd ssh не может определить пользователя,
#    поэтому создаём одну строку root; остальное borg восстанавливает из архива
mkdir /etc
echo 'root:x:0:0:root:/root:/bin/bash' > /etc/passwd
echo 'root:x:0:' > /etc/group
cd / && borg extract --list ::"$ARCHIVE" 2>&1 | tee /root/borg-restore.log

# 5. Проверить
(cd / && find etc -type f -print0 | sort -z | xargs -0 md5sum) > /root/etc-after.md5
diff /root/etc-before.md5 /root/etc-after.md5 && echo "OK: /etc restored, all files identical"

# 6. Запустить бэкап обратно
systemctl daemon-reload && systemctl start borg-backup.timer
rm -rf /etc.old
```

Результат: восстановлены все 704 файла `/etc` из архива `etc-2026-10-16_13:40:00`, контрольные суммы совпадают с исходными, система и бэкап работают. Лог восстановления: [logs/borg-restore.log](logs/borg-restore.log). Первый бэкап после восстановления (`etc-2026-10-16_13:50:26`) прошёл с `exit code: 0`: [logs/borg-backup.log](logs/borg-backup.log).

## Скриншоты

Процесс выполнения задания зафиксирован в [screenshots/](screenshots/).
