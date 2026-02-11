# MTProto Proxy для Telegram — инструкция

Установка MTProto-прокси на VPS с Ubuntu 22.04 или 24.04. Используется реализация [alexbers/mtprotoproxy](https://github.com/alexbers/mtprotoproxy) (Python).

---

## Требования

- **ОС:** Ubuntu 22.04 или 24.04
- **Права:** root (или `sudo`)
- **Место на диске:** не менее 100 МБ
- **Сеть:** сервер с белым IP (VPS)

---

## Быстрая установка (одной командой)

Подключитесь к серверу по SSH и выполните:

```bash
curl -s https://your-domain.com/install.sh | sudo bash
```

Замените `https://your-domain.com/install.sh` на реальный URL скрипта, если вы разместили его на своём домене.

**Локальная установка** (если скрипт уже на сервере):

```bash
sudo bash install.sh
```

Установка полностью автоматическая, ввод с клавиатуры не требуется.

---

## Установка с параметрами

### Задать порт и AD-тег

```bash
PORT=9123 AD_TAG="ваш-тег-от-MTProxybot" sudo -E bash install.sh
```

- **PORT** — порт прокси (9000–9999). Если не указан, выбирается случайный свободный.
- **AD_TAG** — тег рекламы из [@MTProxybot](https://t.me/MTProxybot) в Telegram (для статистики и монетизации).

### Включить автоматическое обновление

Раз в неделю будут обновляться пакеты и код прокси, сервис перезапускаться при необходимости:

```bash
sudo bash install.sh --enable-auto-update
```

---

## После установки

В конце установки скрипт выведет:

- **Ссылку для Telegram** вида:  
  `tg://proxy?server=IP&port=ПОРТ&secret=ee...`  
  Эту ссылку можно отправить пользователям или добавить в бота.
- **Секретный ключ** — храните в надёжном месте.
- **Порт** — по нему прокси доступен снаружи.
- **Команды** для проверки сервиса и логов.

**Пример подключения в Telegram:**

1. Откройте ссылку `tg://proxy?server=...` в браузере или из другого приложения.
2. Или: Настройки → Данные и память → Использовать прокси → Добавить прокси → MTProto — вставьте IP, порт и секрет.

---

## Управление сервисом

| Действие        | Команда |
|-----------------|--------|
| Статус          | `sudo systemctl status mtprotoproxy` |
| Запуск          | `sudo systemctl start mtprotoproxy`  |
| Остановка       | `sudo systemctl stop mtprotoproxy`    |
| Перезапуск      | `sudo systemctl restart mtprotoproxy` |
| Логи в реальном времени | `journalctl -u mtprotoproxy -f` |

---

## Проверка работы (health check)

Проверить, что порт открыт и прокси слушает:

```bash
nc -zv ВАШ_IP ПОРТ
```

Например:

```bash
nc -zv 1.2.3.4 9123
```

Успешный вывод: `Connection to 1.2.3.4 port 9123 [tcp/*] succeeded!`

Эту проверку можно использовать в системах мониторинга (Zabbix, Prometheus и т.п.).

---

## Повторный запуск (идемпотентность)

Скрипт можно запускать несколько раз:

- При повторном запуске **не перезаписываются** существующие порт и секрет в `/opt/mtprotoproxy/config.py`.
- Обновляются сервис systemd, UFW и logrotate.
- Ссылка и секрет в выводе берутся из текущего конфига.

Чтобы сменить порт или секрет, измените конфиг вручную или удалите прокси и установите заново (см. раздел про удаление).

---

## Файлы и каталоги

| Путь | Назначение |
|------|------------|
| `/opt/mtprotoproxy/` | Код и конфиг прокси |
| `/opt/mtprotoproxy/config.py` | Конфигурация (порт, секрет, AD_TAG). Права: 600, владелец `mtproto-proxy` |
| `/etc/systemd/system/mtprotoproxy.service` | Юнит systemd |
| `/etc/logrotate.d/mtprotoproxy` | Ротация логов |
| `/etc/cron.weekly/mtprotoproxy-update` | Скрипт еженедельного обновления (если включён `--enable-auto-update`) |

---

## Решение проблем

### Ошибка: «Must run as root»

Запускайте установку с правами root:

```bash
sudo bash install.sh
```

### Ошибка: «Unsupported OS»

Скрипт рассчитан на Ubuntu. На других дистрибутивах (Debian, CentOS и т.д.) возможны сбои — используйте Ubuntu 22.04/24.04.

### «Proxy may not be listening yet»

Подождите 5–10 секунд и проверьте:

```bash
sudo systemctl status mtprotoproxy
journalctl -u mtprotoproxy -n 50
```

Убедитесь, что порт не занят другим процессом: `ss -tlnp | grep ПОРТ`.

### Telegram не подключается по ссылке

1. Проверьте, что на VPS открыт нужный порт в файрволе (UFW):  
   `sudo ufw status`
2. Убедитесь, что облачный фаервол (панель хостинга) тоже разрешает входящий трафик на этот порт.
3. Проверьте доступность порта с другого хоста:  
   `nc -zv IP_СЕРВЕРА ПОРТ`
4. Убедитесь, что в ссылке или в настройках прокси указаны правильные IP, порт и **полный** секрет (с префиксом `ee` или `dd`).

### Сменить порт или секрет после установки

1. Остановите сервис:  
   `sudo systemctl stop mtprotoproxy`
2. Отредактируйте `/opt/mtprotoproxy/config.py`: измените `PORT` и/или значение в `USERS["tg"]` (32 символа hex).
3. Обновите UFW при смене порта:  
   `sudo ufw allow НОВЫЙ_ПОРТ/tcp`  
   при необходимости удалите старое правило.
4. Запустите сервис:  
   `sudo systemctl start mtprotoproxy`  
   Новую ссылку сформируйте вручную, подставив IP, новый порт и новый секрет (префикс `ee` + 32 hex или `dd` + 32 hex).

### Полное удаление прокси

Если у вас есть скрипт `uninstall.sh`:

```bash
sudo bash uninstall.sh
```

Вручную:

```bash
sudo systemctl stop mtprotoproxy
sudo systemctl disable mtprotoproxy
sudo rm -f /etc/systemd/system/mtprotoproxy.service
sudo rm -f /etc/logrotate.d/mtprotoproxy
sudo rm -f /etc/cron.weekly/mtprotoproxy-update
sudo rm -rf /opt/mtprotoproxy
sudo userdel mtproto-proxy 2>/dev/null || true
sudo systemctl daemon-reload
# При необходимости удалите правило UFW для порта прокси
```

---

## Безопасность и ограничения

- Прокси слушает на всех интерфейсах (`0.0.0.0`). Доступ снаружи ограничивается файрволом (UFW и панель хостинга).
- Конфиг хранится с правами `600`, сервис запускается от пользователя `mtproto-proxy` без лишних привилегий.
- В systemd включены ограничения: `PrivateTmp`, `NoNewPrivileges`, `ProtectSystem`, `ProtectHome` и др.

**Важно:** использование и распространение прокси должно соответствовать правилам Telegram и законодательству вашей страны. Скрипт и эта инструкция не призывают к их нарушению.

---

## Краткая шпаргалка команд

```bash
# Установка (порт и тег по желанию)
curl -s URL_СКРИПТА | sudo bash
PORT=9123 AD_TAG="тег" sudo -E bash install.sh
sudo bash install.sh --enable-auto-update

# После установки
sudo systemctl status mtprotoproxy
journalctl -u mtprotoproxy -f
nc -zv ВАШ_IP ПОРТ
```

Если скрипт выложен у вас на домене, подставьте вместо `URL_СКРИПТА` ваш URL (например, `https://example.com/install.sh`).
