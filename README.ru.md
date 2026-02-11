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

### Включить SOCKS5 (рекомендуется для видео)

Через MTProto видео иногда грузится медленно. SOCKS5 на том же сервере часто даёт более стабильную загрузку медиа в Telegram.

Установка с SOCKS5 (порт по умолчанию 1080 или свой):

```bash
sudo bash install.sh --socks5
# или свой порт:
SOCKS5_PORT=1080 sudo -E bash install.sh --socks5
```

После установки в выводе появится блок **SOCKS5 proxy**: IP, порт. По умолчанию — без логина и пароля. В Telegram: **Настройки → Прокси → SOCKS5** — укажите сервер и порт; при включённой авторизации введите логин и пароль.

**Авторизация SOCKS5 (логин/пароль):**

- Свои логин и пароль (задаются до установки):
  ```bash
  SOCKS5_USER=myuser SOCKS5_PASS=mypass sudo -E bash install.sh --socks5
  ```
- Случайный пароль (логин `socks5`, пароль выведен в конце):
  ```bash
  sudo bash install.sh --socks5 --socks5-auth
  ```
Создаётся системный пользователь; Dante проверяет логин/пароль через PAM. В Telegram при добавлении SOCKS5-прокси укажите этот логин и пароль.

**Важно:** в Dante для авторизации меняется только **socksmethod: username**. Параметр **clientmethod** должен оставаться **none** (значение `username` для clientmethod в Dante не поддерживается и приводит к ошибке запуска).

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

- **MTProto** (ссылка из вывода скрипта): откройте `tg://proxy?server=...` или Настройки → Прокси → MTProto — IP, порт, секрет.
- **SOCKS5** (если установлен `--socks5`): Настройки → Прокси → SOCKS5 — укажите IP и порт (например 1080). Если при установке была включена авторизация (`SOCKS5_USER`/`SOCKS5_PASS` или `--socks5-auth`) — введите выданный логин и пароль. Удобно для просмотра видео.

---

## Управление сервисом

| Действие        | Команда |
|-----------------|--------|
| MTProto: статус | `sudo systemctl status mtprotoproxy` |
| MTProto: логи   | `journalctl -u mtprotoproxy -f` |
| SOCKS5: статус  | `sudo systemctl status danted` |
| SOCKS5: перезапуск | `sudo systemctl restart danted` |

**Если SOCKS5 не подключается:** проверьте интерфейс в `external:` (часто `ens3`, а не `eth0`) и что в конфиге есть `clientmethod: none` и `socksmethod: none` (без авторизации) или только `socksmethod: username` при авторизации — **clientmethod** всегда оставляйте **none**. См. раздел «SOCKS5 не работает» ниже.

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

## Статистика в логах: «0 connects»

В логах сервиса каждые ~10 минут выводится строка вида:

```text
Stats for ... tg: 0 connects (0 current), 0.00 MB, 0 msgs
```

**Почему может быть 0 подключений, хотя Telegram работает:**

1. **В клиенте не выбран прокси**  
   Настройки → Данные и память → Прокси — должен быть **включён** переключатель «Использовать прокси» и выбран именно ваш MTProxy. Иначе приложение может ходить в сеть напрямую, а ссылку при этом открывать.

2. **Трафик идёт не через прокси**  
   Отправьте несколько сообщений, подождите 10–15 минут и снова посмотрите логи:  
   `journalctl -u mtprotoproxy -n 20`  
   Если счётчик `connects` так и остаётся 0, клиент, скорее всего, не использует прокси (проверьте пункт 1).

3. **Краткие сессии**  
   Статистика считает установленные соединения. Очень короткие подключения могли не попасть в последний интервал.

**Как убедиться, что Telegram реально использует прокси:** в настройках прокси в приложении должно быть видно, что прокси активен (например, отображается как «Подключён» или с зелёной меткой). После этого отправьте сообщение и при следующей записи Stats в логах должны появиться ненулевые значения.

**Предупреждение про TLS_DOMAIN** в логах можно убрать, добавив в `/opt/mtprotoproxy/config.py` строку (от имени root):

```python
TLS_DOMAIN = "www.cloudflare.com"
```

После правки перезапустите сервис: `sudo systemctl restart mtprotoproxy`.

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
| `/etc/danted.conf` | Конфиг SOCKS5 (Dante); при авторизации: `socksmethod: username`, `clientmethod: none` |

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

### SOCKS5 не работает

1. **Проверьте сервис:**  
   `sudo systemctl status danted`  
   Если `failed` или `inactive`, смотрите логи: `journalctl -u danted -n 30`.

2. **Правильный конфиг** `/etc/danted.conf`:
   - **Интерфейс** в `external:` — тот, через который сервер выходит в интернет (часто не `eth0`, а `ens3` или `ens5`). Узнать:  
     `ip route get 8.8.8.8 | awk '{print $5; exit}'`
   - **Без авторизации:** глобально должны быть `clientmethod: none` и `socksmethod: none`.
   - **С авторизацией (логин/пароль):** только `socksmethod: username`; **clientmethod обязательно оставьте none** — в Dante значение `username` для clientmethod не поддерживается (ошибка «method username is not a valid»).

   **Пример конфига без авторизации** (подставьте свой интерфейс и порт):

   ```bash
   IFACE=$(ip route get 8.8.8.8 | awk '{print $5; exit}')
   sudo tee /etc/danted.conf << EOF
   logoutput: syslog
   user.privileged: root
   user.unprivileged: nobody
   internal: 0.0.0.0 port = 1080
   external: $IFACE
   clientmethod: none
   socksmethod: none
   client pass { from: 0.0.0.0/0 to: 0.0.0.0/0 log: error }
   socks pass { from: 0.0.0.0/0 to: 0.0.0.0/0 log: error }
   EOF
   sudo systemctl restart danted
   sudo systemctl status danted
   ```

   **Включить авторизацию на уже установленном SOCKS5:**

   ```bash
   # 1) Создать пользователя и пароль
   sudo useradd -r -s /bin/false myuser
   echo "myuser:YourPassword" | sudo chpasswd

   # 2) В конфиге изменить ТОЛЬКО socksmethod (clientmethod оставить none)
   sudo sed -i 's/socksmethod: none/socksmethod: username/' /etc/danted.conf

   # 3) Перезапустить Dante
   sudo systemctl restart danted
   sudo systemctl status danted
   ```

   В Telegram в настройках SOCKS5 укажите логин `myuser` и пароль `YourPassword`.

3. **Проверка с сервера:**  
   Без авторизации:  
   `curl -x socks5://127.0.0.1:1080 -s -o /dev/null -w "%{http_code}" https://api.telegram.org`  
   С авторизацией:  
   `curl -x socks5://ЛОГИН:ПАРОЛЬ@127.0.0.1:1080 -s -o /dev/null -w "%{http_code}" https://api.telegram.org`  
   Должно вывести `200`, `302` или другой код ответа (не ошибку соединения).

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
# Если ставили SOCKS5: sudo systemctl stop danted && sudo apt remove -y dante-server
# При необходимости удалите правила UFW для портов прокси
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

# SOCKS5 без авторизации
sudo bash install.sh --socks5

# SOCKS5 с авторизацией (свой или случайный логин/пароль)
SOCKS5_USER=myuser SOCKS5_PASS=mypass sudo -E bash install.sh --socks5
sudo bash install.sh --socks5 --socks5-auth

# После установки
sudo systemctl status mtprotoproxy
sudo systemctl status danted
journalctl -u mtprotoproxy -f
nc -zv ВАШ_IP ПОРТ
```

Если скрипт выложен у вас на домене, подставьте вместо `URL_СКРИПТА` ваш URL (например, `https://example.com/install.sh`).
