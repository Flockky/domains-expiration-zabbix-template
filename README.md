
# Zabbix Template: Domain Name Expiration Monitor

Шаблон для мониторинга сроков истечения регистрации доменных имен в Zabbix 7.4+.
Использует LLD (Low-Level Discovery) для автоматического обнаружения доменов из списка и проверки их статуса через `whois`.

## Возможности

*   **Автоматическое обнаружение:** Домены загружаются из JSON-файла на сервере Zabbix.
*   **Универсальный парсинг:** Скрипт поддерживает различные форматы дат истечения (`Registry Expiry Date`, `paid-till`, `Expiration Date` и др.).
*   **Оповещения:**
    *   Триггер **HIGH**: Срок действия истекает менее чем через 14 дней.
    *   Триггер **HIGH**: Не удалось получить данные whois (ошибка парсинга или сети).
*   **Гибкость:** Легко добавлять новые домены редактированием одного файла.

## Требования

1.  Zabbix Server / Proxy версии **7.4** или выше.
2.  Установленная утилита `whois` на хосте, где выполняется скрипт.
    ```bash
    # Для RHEL/CentOS/Rocky
    yum install whois -y
    
    # Для Debian/Ubuntu
    apt-get install whois -y
    ```
3.  Разрешение выполнения команд `system.run` в конфигурации Zabbix Agent (`EnableRemoteCommands=1`) или на стороне сервера, в зависимости от вашей архитектуры.

## Установка

### 1. Подготовка скриптов и файлов

Создайте директорию для внешних скриптов (если её нет) и разместите там файлы.
Стандартный путь в шаблоне: `/usr/lib/zabbix/externalscripts/`

#### Шаг 1.1: Файл со списком доменов
Создайте файл `DomainExpireList` в формате JSON массива объектов.

Путь: `/usr/lib/zabbix/externalscripts/DomainExpireList`

Пример содержимого (`DomainExpireList`):
```json
[
  {
    "Domain": "example.com"
  },
  {
    "Domain": "mydomain.ru"
  }
]
```
> **Важно:** Убедитесь, что файл имеет корректные права доступа для чтения пользователем `zabbix`.
> ```bash
> chown zabbix:zabbix /usr/lib/zabbix/externalscripts/DomainExpireList
> chmod 644 /usr/lib/zabbix/externalscripts/DomainExpireList
> ```

#### Шаг 1.2: Скрипт проверки Whois
Скачайте или создайте скрипт `Getwhois.sh`.

Путь: `/usr/lib/zabbix/externalscripts/Getwhois.sh`

```bash
#!/bin/bash
DOMAIN="$1"
# Получаем WHOIS и сохраняем во временный файл
WHOIS_OUTPUT=$(mktemp)
trap "rm -f $WHOIS_OUTPUT" EXIT
whois "$DOMAIN" > "$WHOIS_OUTPUT"

# Функция извлечения даты в Unix timestamp
get_expiry_date() {
    local FILE="$1"
    # Пытаемся найти дату разными способами
    if grep -q 'Registry Expiry Date:' "$FILE"; then
        awk -F': ' '/Registry Expiry Date:/ {print $2}' "$FILE"
    elif grep -q 'paid-till' "$FILE"; then
        awk -F': ' '/paid-till/ {print $2}' "$FILE"
    elif grep -q 'Expiration Date:' "$FILE"; then
        awk -F': ' '/Expiration Date:/ {print $2}' "$FILE"
    elif grep -q 'Expiry date:' "$FILE"; then
        awk '/Expiry date:/ {print $3}' "$FILE"
    else
        echo "Ошибка: не найдено подходящее поле с датой истечения" >&2
        exit 1
    fi
}

# Извлекаем значение даты
EXPIRY_DATE=$(get_expiry_date "$WHOIS_OUTPUT")

# Убираем миллисекунды, если есть
EXPIRY_DATE_CLEAN=$(echo "$EXPIRY_DATE" | cut -d '.' -f1)

# Преобразуем в Unix timestamp
TIMESTAMP=$(date -u +%s -d "$EXPIRY_DATE_CLEAN" 2>/dev/null)

if [[ $? -ne 0 ]]; then
    echo "Ошибка: не удалось распознать формат даты '$EXPIRY_DATE_CLEAN'" >&2
    exit 1
fi

# Выводим результат
echo "$TIMESTAMP"
```

Сделайте скрипт исполняемым:
```bash
chmod +x /usr/lib/zabbix/externalscripts/Getwhois.sh
chown zabbix:zabbix /usr/lib/zabbix/externalscripts/Getwhois.sh
```

### 2. Импорт шаблона

1.  Скачайте файл шаблона `template_domain_expire.yaml` из этого репозитория.
2.  Перейдите в Zabbix Web Interface: **Data collection** -> **Templates**.
3.  Нажмите **Import**.
4.  Выберите файл и нажмите **Import**.
5.  Шаблон появится в группе **Наши шаблоны**.

### 3. Назначение шаблона хосту

1.  Создайте новый хост или выберите существующий (обычно это сам Zabbix Server или хост, где установлен агент с доступом к `system.run`).
2.  Перейдите во вкладку **Templates**.
3.  Добавьте шаблон **Срок действия доменов** (`Domain name expire`).
4.  Убедитесь, что на хосте разрешены удаленные команды (`system.run`), если вы используете эту архитектуру.

## Настройка оповещений

В шаблоне уже настроены триггеры. Для корректной работы уведомлений убедитесь, что:
1.  У пользователей или групп пользователей настроены медиа (Email, Telegram, Slack и т.д.).
2.  Действия (Actions) или Процессы оповещений (Alerting processes) в Zabbix 7.4 настроены на обработку триггеров с соответствующими тегами или из нужных групп хостов.

## Troubleshooting

### Тестирование скрипта вручную
Выполните от имени пользователя `zabbix`:
```bash
sudo -u zabbix /usr/lib/zabbix/externalscripts/Getwhois.sh google.com
```
Ожидаемый результат: число (Unix timestamp), например `1798765432`.

### Ошибка "Permission denied"
Проверьте права на файлы в `/usr/lib/zabbix/externalscripts/` и наличие бита исполнения у `.sh` файла.

### SELinux
Если используется SELinux и команды `system.run` не выполняются или `whois` не имеет доступа к сети:
1.  Проверьте логи: `/var/log/audit/audit.log`.
2.  Возможно, потребуется настроить политики SELinux.
3.  Для доступа процессов веб-сервера/агента к сети часто требуется:
    ```bash
    setsebool -P httpd_can_network_connect 1
    ```

### Данные не обновляются
1.  Проверьте элемент данных "Получение списка доменов". Он должен возвращать валидный JSON.
2.  Проверьте права на чтение файла `DomainExpireList` пользователем `zabbix`.
3.  Посмотрите последние значения элемента данных и логгируемые ошибки в Zabbix Frontend.
