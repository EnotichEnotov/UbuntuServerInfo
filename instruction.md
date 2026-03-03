# Настройка Firebird (Ubuntu) БД + Windows client по TCP (порт 3050)

Цель: программа на Windows подключается к Firebird на Ubuntu по TCP (провод/локалка).  
Сценарий: программа сначала создаёт `.FDB` локально на Windows → переносим
файл на Ubuntu → прописываем путь в программе → работа идёт по TCP.

---

## Часть 1. Подготовка Ubuntu (сервер базы)

### 1) Узнать IP Ubuntu-сервера (он нужен Windows-клиенту)
На Ubuntu:
```bash
hostname -I
ip -4 a
```
Пример результата: `192.168.1.40` (дальше в инструкции использую этот IP).

---

### 2) Установить Firebird 3.0 (сервер + утилиты)
```bash
sudo apt update
sudo apt install -y firebird3.0-server firebird3.0-utils
```

Проверка, что утилиты есть:
```bash
which isql-fb
which gsec
```

---

### 3) Включить автозапуск Firebird при перезагрузке
```bash
sudo systemctl enable --now firebird3.0
sudo systemctl status firebird3.0 --no-pager
```

---

### 4) Сделать так, чтобы Firebird слушал порт 3050 **наружу** (не только localhost)
#### 4.1) Проверка, где слушает порт 3050 сейчас
```bash
sudo ss -lntp 'sport = :3050'
```
Нужно увидеть `*:3050` или `0.0.0.0:3050` (или конкретный IP сервера, например `192.168.1.40:3050`).  
Если видишь **только** `127.0.0.1:3050` — удалённый доступ не заработает.

#### 4.2) Настройка bind-адреса Firebird (Firebird 3.0)
Открой конфиг:
```bash
sudo nano /etc/firebird/3.0/firebird.conf
```

Найди параметр `RemoteBindAddress`.  
Сделай одно из двух (любой вариант норм):

- слушать на всех интерфейсах:
  ```
  RemoteBindAddress = 0.0.0.0
  ```
- или слушать только на IP сервера(пк с убунтой) (безопаснее):
  ```
  RemoteBindAddress = 192.168.1.40
  ```

Сохраните, перезапустите:
```bash
sudo systemctl restart firebird3.0
sudo ss -lntp 'sport = :3050'
```

---

### 5) Открыть порт 3050 в UFW (если UFW включён)
Проверка:
```bash
sudo ufw status
```
Если `Status: active`, разрешите доступ **только из локальной подсети** (пример для 192.168.1.0/24):
```bash
sudo ufw allow from 192.168.1.0/24 to any port 3050 proto tcp
sudo ufw status
```

---

### 6) Создать папку под базы и выставить права
```bash
sudo mkdir -p /var/lib/firebird/Profsegment
sudo chown -R firebird:firebird /var/lib/firebird/Profsegment
sudo chmod 750 /var/lib/firebird/Profsegment
```

---

### 7) Задать пароль SYSDBA = masterkey (стандартный)
На Ubuntu:
```bash
sudo dpkg-reconfigure firebird3.0-server
```
Когда спросит пароль SYSDBA — введите: `masterkey`.

---

## Часть 2. Проверка Windows (клиент)

### 1) Проверить, что порт 3050 доступен по сети с Windows (если следующие шаги не будут работать)
PowerShell:
```powershell
Test-NetConnection 192.168.1.40 -Port 3050
```
Нужно: `TcpTestSucceeded : True`.

Или через cmd:
```
telnet 192.168.1.40 3050
```

---

## Часть 3. Перенос `.FDB` с Windows на Ubuntu

⚠️ Важно: перед копированием **закрой программу**, чтобы файл `.FDB` не был открыт/изменялся во время копирования.

### 1) На Windows: скопировать `.FDB` на Ubuntu через SCP (CMD)
Пример:
```cmd
scp "D:\ProfSegment\BASE-3-2703.FDB" enotichenotov@192.168.1.40:/tmp/
```

---

### 2) На Ubuntu: переместить файл в папку Firebird и выставить права
```bash
sudo mv "/tmp/BASE-3-2703.FDB" "/var/lib/firebird/Profsegment/"
sudo chown firebird:firebird "/var/lib/firebird/Profsegment/BASE-3-2703.FDB"
sudo chmod 660 "/var/lib/firebird/Profsegment/BASE-3-2703.FDB"
ls -lah /var/lib/firebird/Profsegment
```

---
### 3) Проверка подключения из Windows через isql (TCP)
```cmd
"C:\Program Files\Firebird\Firebird_3_0\isql.exe" -user SYSDBA -password masterkey 192.168.1.40/3050:/var/lib/firebird/Profsegment/BASE-3-2703.FDB
```

---

## Часть 4. Итоговые параметры подключения для программы на Windows

### 1) Итоговый адрес базы (то, что использует клиент по TCP)
Полный формат (часто принимается как “Database”):
```
192.168.1.40/3050:/var/lib/firebird/Profsegment/BASE-3-2703.FDB
```

### 2) Учётка Firebird (по умолчании в программе Профстрой)
- user: `SYSDBA`
- password: `masterkey`

---

## Часть 5. Финальные проверки “программа реально подключилась”

### 1) Проверка с Windows через isql
```cmd
"C:\Program Files\Firebird\Firebird_3_0\isql.exe" -user SYSDBA -password masterkey 192.168.1.40/3050:/var/lib/firebird/Profsegment/BASE-3-2703.FDB
```
Внутри `SQL>`:
```sql
select current_user from rdb$database;
select rdb$get_context('SYSTEM','ENGINE_VERSION') from rdb$database;
quit;
```

### 2) Проверка на Ubuntu: есть ли активные TCP-соединения к 3050 (когда программа запущена)
```bash
sudo ss -tnp | grep ':3050'
```

### 3) Проверка на Ubuntu: открыт ли файл базы процессом Firebird (когда есть подключения)
```bash
sudo lsof /var/lib/firebird/Profsegment/BASE-3-2703.FDB
```
Пусто = сейчас никто не подключён.  
Есть строки = есть активные подключения (это нормально, если программа работает).

---

## Частая ошибка (важно)
Если `isql.exe` запущен на Windows, нельзя делать:
```sql
connect "/var/lib/firebird/....FDB"
```
Это заставит Windows пытаться открыть Linux-путь как локальный файл и даст ошибку.

На Windows всегда используй формат:
```
IP/3050:/var/...
```
