# Синхронизация структуры подразделений (OU) из Active Directory в Яндекс 360

Скрипт `sync_ou_for_users.py` выполняет одностороннюю синхронизацию иерархии организационных единиц (OU) из локальной Active Directory в подразделения Яндекс 360 через API. Пользователи, найденные в указанных OU, автоматически распределяются по соответствующим подразделениям в Яндекс 360 на основе совпадения email-адреса (nickname или alias).

## Параметры в файле .env

Файл `.env` должен находиться в одном каталоге со скриптом.

| Параметр | Описание |
|---|---|
| `OAUTH_TOKEN` | OAuth-токен для доступа к API Яндекс 360 |
| `ORG_ID` | Идентификатор организации в Яндекс 360 |
| `LDAP_HOST` | Адрес контроллера домена (DNS-имя или IP). При нескольких доменах в лесу AD — обязательно DNS-имя |
| `LDAPS_ENABLED` | Использовать LDAPS (`True`/`False`). При `True` порт должен быть `636` |
| `LDAP_PORT` | Порт подключения: `389` для LDAP одного домена, `3268` для Global Catalog, `636` для LDAPS |
| `LDAP_USER` | Учётная запись для подключения к LDAP в формате `DOMAIN\username` |
| `LDAP_PASSWORD` | Пароль учётной записи |
| `LDAP_BASE_DN` | Корневой DN для поиска пользователей, например `DC=company,DC=local` |
| `LDAP_SEARCH_FILTER` | LDAP-фильтр для выборки пользователей, которые синхронизированы в Яндекс 360. Например: `(memberOf=CN=Yandex360,OU=Groups,DC=company,DC=local)` |
| `ATTRIB_LIST` | Список атрибутов LDAP через запятую: `distinguishedName,mail,displayName,department,objectCategory,sAMAccountName,cn` |
| `ROOT_OU` | Корневые OU, из которых строится иерархия. Несколько OU разделяются точкой с запятой: `"OU=Office,DC=company,DC=local;OU=Branch,DC=company,DC=local"` |
| `AD_DEPS_OUT_FILE` | Имя файла для сохранения промежуточных данных об иерархии подразделений |
| `AD_DATA_OUT_FILE` | Имя файла для сохранения данных AD (пользователи + OU) |
| `API_DATA_OUT_FILE` | Имя файла для сохранения состояния подразделений в Яндекс 360 (до и после синхронизации) |
| `DRY_RUN` | Режим тестового прогона (`True`/`False`). При `True` никакие изменения в Яндекс 360 не вносятся, но все действия логируются |
| `LOAD_AD_DATA_FROM_FILE` | Загружать данные AD из файла `AD_DATA_OUT_FILE` вместо подключения к LDAP (`True`/`False`). Удобно для отладки |
| `KEEP_EMPTY_EXTERNAL_ID_IN_Y360` | Не удалять из Яндекс 360 подразделения, у которых не задан `externalId` (`True`/`False`). Значение `True` позволяет сохранить вручную созданные подразделения |

## Получение OAuth-токена

1. Перейдите на [https://oauth.yandex.ru](https://oauth.yandex.ru) и авторизуйтесь от имени администратора организации Яндекс 360.
2. Нажмите **Создать приложение** (или откройте существующее).
3. Заполните название приложения (произвольное).
4. В разделе **Платформы** выберите **Веб-сервисы** и укажите Callback URL: `https://oauth.yandex.ru/verification_code`.
5. В разделе **Права** добавьте доступ к API Яндекс 360:
   - `directory:read_departments` — чтение списка подразделений
   - `directory:write_departments` — создание/изменение/удаление подразделений
   - `directory:read_users` — чтение списка пользователей
   - `directory:write_users` — изменение пользователей (перемещение между подразделениями)
6. Сохраните приложение. Скопируйте **ClientID**.
7. Откройте в браузере URL для получения токена:
   ```
   https://oauth.yandex.ru/authorize?response_type=token&client_id=ВАШ_CLIENT_ID
   ```
8. После подтверждения разрешений вы будете перенаправлены на страницу с токеном. Скопируйте значение `access_token` и вставьте в параметр `OAUTH_TOKEN` в файле `.env`.

> Токен имеет ограниченный срок действия. При истечении срока повторите шаг 7.

## Настройка окружения и запуск

### Требования

- Python 3.9+
- Сетевой доступ к контроллеру домена AD (порты 389/3268/636)
- Сетевой доступ к `api360.yandex.net` (HTTPS, порт 443)

### Установка

```bash
# Клонирование или копирование скрипта в рабочий каталог
cd /path/to/sync_ou_from_ad

# Создание виртуального окружения
python -m venv venv

# Активация окружения
# Linux / macOS:
source venv/bin/activate
# Windows (cmd):
venv\Scripts\activate.bat
# Windows (PowerShell):
venv\Scripts\Activate.ps1

# Установка зависимостей
pip install -r requirements.txt
```

### Запуск

```bash
# Активация окружения (если не активировано)
# Linux / macOS:
source venv/bin/activate
# Windows:
venv\Scripts\activate.bat

# Запуск скрипта
python sync_ou_for_users.py
```

## Сценарии использования

### 1. Первый запуск — тестовый прогон (DRY_RUN)

Установите `DRY_RUN = True` в `.env`. Скрипт подключится к AD и Яндекс 360, построит план изменений и запишет его в лог, но не выполнит ни одного изменения. Проверьте файл `sync_ou.log` и убедитесь, что запланированные действия корректны.

### 2. Полная синхронизация

Установите `DRY_RUN = False`. Скрипт:
- прочитает иерархию OU из AD, начиная с корневых OU, указанных в `ROOT_OU`;
- сопоставит её с текущими подразделениями в Яндекс 360;
- создаст недостающие подразделения, сохраняя вложенность;
- удалит подразделения из Яндекс 360, которых больше нет в AD (при этом пользователи из них будут перемещены в подразделение «Все»);
- распределит пользователей по подразделениям на основе совпадения email (nickname или alias).

### 3. Отладка без подключения к AD

Установите `LOAD_AD_DATA_FROM_FILE = True`. Скрипт загрузит данные из файла `AD_DATA_OUT_FILE` (создаётся при обычном запуске) вместо подключения к LDAP. Удобно для повторных тестов и отладки маппинга пользователей.

### 4. Сохранение вручную созданных подразделений

Установите `KEEP_EMPTY_EXTERNAL_ID_IN_Y360 = True`. Подразделения в Яндекс 360, не имеющие `externalId` (т.е. созданные вручную, а не через синхронизацию), не будут удалены.

## Пример иерархии OU

### Структура в Active Directory

```
DC=company,DC=local
├── OU=Office
│   ├── OU=Users
│   │   ├── user: Иван Иванов (ivan@company.ru)
│   │   ├── user: Petr Petrov (petrov@company.ru)
│   │   ├── user: Никита Кравцов (nikita@company.ru)
│   │   ├── user: Марина Егорова (marina@company.ru)
│   │   ├── OU=Branch1
│   │   │   ├── user: Андрей Шатров (shatrov@company.ru)
│   │   │   └── OU=Deps1
│   │   │       └── user: Евгений Коротов (korotov@company.ru)
│   │   └── OU=Branch2
│   │       ├── OU=Deps1
│   │       │   └── user: Роман Светлов (svetlov@company.ru)
│   │       └── OU=Deps2
│   ├── OU=Contacts
│   │   └── user: Александр Морозов (morozov@company.ru)
│   ├── OU=Computers
│   ├── OU=Groups
│   └── OU=Test
└── OU=Head
```

В параметре `ROOT_OU` указаны корневые OU: `"OU=HAB,DC=company,DC=local;OU=Office,DC=company,DC=local"`.

Параметр `LDAP_SEARCH_FILTER` определяет, какие пользователи попадут в синхронизацию (например, участники группы `Yandex360`).

### Результат в Яндекс 360

```
Все (корневое подразделение, id=1)
├── Head
├── Office
│   ├── Users
│   │   ├── Иван Иванов (ivan@company.ru)
│   │   ├── Petr Petrov (petrov@company.ru)
│   │   ├── Никита Кравцов (nikita@company.ru)
│   │   ├── Марина Егорова (marina@company.ru)
│   │   ├── Branch1
│   │   │   ├── Андрей Шатров (shatrov@company.ru)
│   │   │   └── Deps1
│   │   │       └── Евгений Коротов (korotov@company.ru)
│   │   └── Branch2
│   │       ├── Deps1
│   │       │   └── Роман Светлов (svetlov@company.ru)
│   │       └── Deps2
│   ├── Contacts
│   │   └── Александр Морозов (morozov@company.ru)
│   ├── Computers
│   ├── Groups
│   └── Test
```

Иерархия подразделений полностью воспроизводится, а пользователи размещаются в тех же подразделениях, что и в AD. Сопоставление пользователей происходит по email (nickname/alias в Яндекс 360 = mail-атрибут в AD). Пользователи, не попавшие ни в одно подразделение, остаются в подразделении «Все».

## Периодический запуск в Windows Task Scheduler

### 1. Создайте .bat-файл для запуска

Создайте файл `run_sync.bat` в каталоге со скриптом:

```bat
@echo off
cd /d "C:\path\to\sync_ou_from_ad"
call venv\Scripts\activate.bat
python sync_ou_for_users.py
```

### 2. Настройте задание в Task Scheduler

1. Откройте **Task Scheduler** (`taskschd.msc`).
2. В правой панели нажмите **Create Task** (Создать задачу).
3. На вкладке **General**:
   - задайте имя, например `Yandex360 OU Sync`;
   - выберите **Run whether user is logged on or not**;
   - установите **Run with highest privileges**.
4. На вкладке **Triggers** нажмите **New** и настройте расписание, например: ежедневно, каждые 2 часа.
5. На вкладке **Actions** нажмите **New**:
   - Action: **Start a program**;
   - Program/script: `C:\path\to\sync_ou_from_ad\run_sync.bat`;
   - Start in: `C:\path\to\sync_ou_from_ad`.
6. На вкладке **Settings**:
   - установите **Stop the task if it runs longer than** `1 hour`;
   - установите **If the task is already running, then the following rule applies**: **Do not start a new instance**.
7. Нажмите **OK** и введите пароль учётной записи Windows.

> Убедитесь, что учётная запись, под которой выполняется задание, имеет сетевой доступ к контроллеру домена AD и к `api360.yandex.net`.

## Поиск информации в файле лога

Скрипт ведёт лог в файл `sync_ou.log` (ротация по размеру, до 10 МБ на файл, до 20 файлов с суффиксами `.1`, `.2` и т.д.).

Формат строки лога:

```
2026-03-09 23:10:36.857 INFO:	Успех - подразделение Computers создано успешно.
```

### Типичные запросы для поиска

**Начало и конец каждого запуска:**
```bash
grep "Запуск скрипта\|End" sync_ou.log
```

**Все ошибки:**
```bash
grep "ERROR" sync_ou.log
```

**Ошибки API (неудачные HTTP-запросы):**
```bash
grep "ОШИБКА" sync_ou.log
```

**Созданные подразделения:**
```bash
grep "создано успешно" sync_ou.log
```

**Удалённые подразделения:**
```bash
grep "удалено успешно" sync_ou.log
```

**Перемещение пользователей по подразделениям:**
```bash
grep "Change department of user" sync_ou.log
```

**Действия в режиме DRY_RUN:**
```bash
grep -i "dry run" sync_ou.log
```

**Поиск информации о конкретном пользователе:**
```bash
grep "petrov@company.ru" sync_ou.log
```

**Подразделения, удалённые из-за отсутствия в AD:**
```bash
grep "not synced from AD" sync_ou.log
```

В PowerShell (Windows) вместо `grep` используйте `Select-String`:

```powershell
Select-String -Path sync_ou.log -Pattern "ERROR"
Select-String -Path sync_ou.log -Pattern "создано успешно"
Select-String -Path sync_ou.log -Pattern "petrov@company.ru"
```
