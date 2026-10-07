# Отчёт по домашней работе

## Проблема 1

### Что было обнаружено

После запуска setup.sh сервис не работает. systemctl status показывает Active: activating (auto-restart), процесс завершается сразу после старта с кодом status=203/EXEC. В ExecStart указан файл /opt/linux-devops-homework/app.py


### Как проводилась диагностика

```bash
systemctl status homework-app.service
ls -l /opt/linux-devops-homework/
```


### В чём была причина

В unit-файле в ExecStart указан несуществующий файл app.py. systemd не может запустить несуществующий файл, поэтому процесс завершался с кодом 203/EXEC.


### Что было изменено

```bash
# systemd/homework-app.service
# было:  ExecStart=/opt/linux-devops-homework/app.py
# стало: ExecStart=/opt/linux-devops-homework/server.py
sudo cp systemd/homework-app.service /etc/systemd/system/homework-app.service
sudo systemctl daemon-reload
sudo systemctl restart homework-app.service
```


### Почему было выбрано это решение

Приложение установлено в /opt/linux-devops-homework/ под именем server.py, и это единственный файл в каталоге. Права на запуск у него есть, поэтому достаточно указать правильный путь в ExecStart. Правку сделала в файле репозитория и скопировала его в систему, чтобы исправление было в Git. После изменения unit-файла нужен daemon-reload, иначе systemd использует старую версию из памяти.

### Как проверялся результат

Ошибка 203/EXEC пропала: в ExecStart теперь server.py, процесс запускается, но завершается с status=1/FAILURE. Значит, программа стартовала и упала уже по своей причине. Это следующая проблема.

```bash
systemctl status homework-app.service
```


---

## Проблема 2

### Что было обнаружено

После запуска homework-app.service сервис не работает. systemctl status показывает Failed to start и status=1/FAILURE, server.py запускается, но в нем ошибка, поэтому он ложится. ВПосле нескольких перезапусков
systemd сдался: Start request repeated too quickly


### Как проводилась диагностика
В журнале сервиса Python выдаёт ошибку: PermissionError: [Errno 13] Permission denied: '/var/lib/linux-devops-homework/startup.log'.
То есть программа пытается открыть на запись файл startup.log в каталоге данных и получает отказ.

ls -ld показал: drwx------ root root /var/lib/linux-devops-homework— владелец root, у группы и остальных прав нет совсем.

```bash
journalctl -u homework-app.service -n 30 --no-pager
ls -ld /var/lib/linux-devops-homework
```


### В чём была причина

У homework нет никаких прав по отношению к этой директории


### Что было изменено

```bash
sudo chown homework:homework /var/lib/linux-devops-homework
sudo chmod 0750 /var/lib/linux-devops-homework
```


### Почему было выбрано это решение

Директория должна иметь все права для пользователя homework, для группы же homework также все, но без прав изменения, остальные не должны иметь к файлу доступа вообще, потому что так треуют условия. Запуск сервиса от root запрещен и не безопасен 

### Как проверялся результат

`ls -ld` показывает drwxr-x--- 2 homework homework 4096 Oct  5 10:03 /var/lib/linux-devops-homework. После перезапуска сервис в состоянии active (running),
ошибки PermissionError в журнале нет.

```bash
ls -ld /var/lib/linux-devops-homework
sudo systemctl restart homework-app.service
systemctl status homework-app.service
```

## Проблема 3

### Что было обнаружено
check.sh сказал, что TCP/8080 is reachable only through loopback и в логе сервиса он был listening on 127.0.0.1:8080, хотя должен был на 0.0.0.0:8080

### Как проводилась диагностика
```bash
ss -lntp
sudo ss -lntp | grep 8080
cat /etc/linux-devops-homework/app.conf
```
у порта 8080 в Local Address было 127.0.0.1:8080, для сравнения SSH слушает 0.0.0.0:22 + с sudo видно процесс и его pid совпадает с main pid

### В чём была причина

В конфиге приложения /etc/linux-devops-homework/app.conf в параметре host был указан адрес 127.0.0.1 (loopback). Поэтому приложение слушало порт 8080 только на внутреннем интерфейсе, и подключиться к нему можно было только с самой машины.


### Что было изменено
```bash
# config/app.conf — было: 
[server]
host = 127.0.0.1
port = 8080
стало: 
[server]
host = 0.0.0.0
port = 8080

sudo cp config/app.conf /etc/linux-devops-homework/app.conf
sudo systemctl restart homework-app.service
```
daemon-reload не нужен, т.к. файл, который исправляла не относится к unit файлу, он конфиг приложения, вместо него сделала restart, его хватило, потому что приложение читает app.conf при запуске

### Почему было выбрано это решение

127.0.0.1:8080 не подходит, потому что это loopback, т.е. мы можем зайти через него только изнутри той же машины, а вот 0.0.0.0 слушает все адреса или интерфейсы машины, значит, подключиться можно и снаружи, поэтому вход должен быть через него 
Правила в репозитории, а потом копировала в /etc, потому что по файлу README исправленные unit-файл и конфигурация приложения находятся в репозитории, а не только в системных каталогах локальной машины.

### Как проверялся результат
```bash
sudo ss -lntp | grep 8080
 sudo bash ./scripts/check.sh
```
теперь ss показывает LISTEN 0      5            0.0.0.0:8080       0.0.0.0:*    users:(("python3",pid=13619,fd=3))
и поменялась строка [PASS] TCP/8080 listens on 0.0.0.0
---

## Проблема 4

### Что было обнаружено
сервис работает, но выходят следующие ошибки:
[FAIL] systemd ExecStart is incorrect
       Next: Inspect: systemctl status homework-app; systemctl cat homework-app; ls -l /opt/linux-devops-homework /opt/linux-devops-homework/app

[FAIL] repository still contains the original broken unit
       Next: Apply the same final unit configuration to systemd/homework-app.service before committing
       
при этом сервис active и /health отвечае

### Как проводилась диагностика
```bash
grep -n "ExecStart" scripts/check.sh
ls -l /opt/linux-devops-homework/
grep -n "server.py" scripts/setup.sh
grep -n "_DIR=" scripts/setup.sh
```

ExecStart ожидает такой путь к файлу if grep -q '^ExecStart=/opt/linux-devops-homework/app/server.py$' 
и server.py сейчас копируется сюда  81:install -m 0755 "${REPO_DIR}/app/server.py" "${APP_DIR}/server.py"
при этом 12:REPO_DIR="$(cd -- "${SCRIPT_DIR}/.." && pwd)"
-rwxr-xr-x 1 root root 1304 Oct  5 10:03 server.py - т.е. показывается только server, без app
скрипты задания расходятся — setup.sh кладёт приложение в одно место,
а check.sh ищет его в другом.

### В чём была причина
setup.sh устанавливает приложение в /opt/linux-devops-homework/server.py, а check.sh ожидает его в /opt/linux-devops-homework/app/server.py. После проблемы 1 unit-файл указывал на фактическое расположение файла, поэтому сервис работал, но не соответствовал требуемому конечному состоянию.

### Что было изменено
Изменения состояния системы (в Git не отражаются):
```bash
sudo mkdir /opt/linux-devops-homework/app
sudo mv /opt/linux-devops-homework/server.py /opt/linux-devops-homework/app/server.py
ls -l /opt/linux-devops-homework/app/
```

Изменения в репозитории:
```bash
# systemd/homework-app.service — было
ExecStart=/opt/linux-devops-homework/server.py
стало: ExecStart=/opt/linux-devops-homework/app/server.py
sudo cp systemd/homework-app.service /etc/systemd/system/homework-app.service
sudo systemctl daemon-reload
sudo systemctl restart homework-app.service
```

### Почему было выбрано это решение

Поменяла систему, а не скриты, потому что по правилам из README нельзя х менять
Использовала mv, а не cp, чтобы в системе осталась одна копия server.py. При копировании было бы два одинаковых файла, и непонятно, какой из них настоящий
mv сохраняет права и владельца файла: server.py остался -rwxr-xr-x root root, поэтому пользователь homework 
по-прежнему может его запускать.
Чтобы дойти до файла, нужно ещё право x на каталог: ls -ld /opt/linux-devops-homework/app показал drwxr-xr-x root root
Расхождение между setup.sh и check.sh  недочёт самого задания? отмечаю его здесь, чтобы было понятно, почему понадобился перенос файла

### Как проверялся результат
```bash
 systemctl status homework-app.service
sudo bash ./scripts/check.sh
```
● homework-app.service - Linux DevOps Homework Service
     Loaded: loaded (/etc/systemd/system/homework-app.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-10-05 17:23:34 UTC; 9s ago
 Invocation: 91d1b7be205940dfb05d3ce89636f8b0
   Main PID: 14862 (python3)
      Tasks: 1 (limit: 1697)
     Memory: 10.2M (peak: 10.2M)
        CPU: 36ms
     CGroup: /system.slice/homework-app.service
             └─14862 python3 /opt/linux-devops-homework/app/server.py

Linux DevOps Homework Checker

[PASS] systemd ExecStart points to the application
[PASS] service runs as homework:homework
[PASS] state directory ownership and permissions are correct
[PASS] service is enabled
[PASS] service is active
[PASS] running process has the expected UID
[PASS] TCP/8080 listens on 0.0.0.0
[PASS] GET /health returns the expected response
[PASS] fixed systemd unit is saved in the repository
[PASS] fixed application configuration is saved in the repository

## Дополнительные наблюдения
### Запуск скриптов задания
sudo ./scripts/setup.sh выдавал Permission denied. ls -l scripts/ показал вывод — права без x, т.е. у setup.sh и check.sh нет права на выполнение. Без бита x файл нельзя запустить как программу даже от root. Менять эти скрипты (в том числе chmod)
запрещено условиями, поэтому запускала их через интерпретатор:
sudo bash ./scripts/setup.sh
sudo bash ./scripts/check.sh 

### Как определить точный ExecStart
systemctl cat homework-app.service показывает unit файл, который использует systemd, а systemctl show -p ExecStart homework-app.serviceb  значение параметра так, как его видит systemd.

### Процесс сервиса

```bash
systemctl show -p MainPID homework-app.service
ps -o pid,ppid,user,group,cmd -p 14862
ps -o pid,comm,cmd -p 1
```

MainPID=14862

    PID    PPID USER     GROUP    CMD
  14862       1 homework homework python3 /opt/linux-devops-homework/app/server.py

    PID COMMAND         CMD
      1 systemd         /usr/lib/systemd/systemd --switched-root --system --deserialize=50


PID процесса сервиса — 14862, PPID — 1. Процесс с PID 1 — это systemd, первый процесс системы: именно он запускает сервис по unit-файлу и следит за ним (перезапускает при падении согласно Restart=on-failure). Поэтому родителем процесса сервиса является systemd.

Процесс работает от пользователя homework и группы homework, а не от root, и запущен именно /opt/linux-devops-homework/app/server.py — то есть приложение из нового расположения после исправления проблемы 4.

### /proc/<PID>

```bash
grep -E "Name|State|PPid|Uid|Gid" /proc/14862/status
tr '\0' ' ' < /proc/14862/cmdline; echo
sudo ls -l /proc/14862/fd/
sudo ss -lntpe | grep 8080
```


Name:   python3
State:  S (sleeping)
PPid:   1
Uid:    999     999     999     999
Gid:    983     983     983     983

python3 /opt/linux-devops-homework/app/server.py

lr-x------ 1 homework homework 64 Oct  5 17:23 0 -> /dev/null
lrwx------ 1 homework homework 64 Oct  5 17:23 1 -> socket:[123936]
lrwx------ 1 homework homework 64 Oct  5 17:23 2 -> socket:[123936]
lrwx------ 1 homework homework 64 Oct  5 17:23 3 -> socket:[123938]


**/proc/14862/status** подтверждает:
- Name: python3 — процесс является интерпретатором Python;
- State: S (sleeping) — процесс жив и спит в ожидании входящих подключений, это нормальное состояние сервера;
- PPid: 1 — родитель systemd (совпадает с выводом ps);
- Uid: 999 и Gid: 983 во всех четырёх полях (реальный, эффективный, сохранённый и файловый идентификаторы) — совпадают с id homework (uid=999(homework) gid=983(homework)).
  Значит, процесс работает от homework и не имеет прав root.

**/proc/14862/cmdline** показывает python3 /opt/linux-devops-homework/app/server.py.
В ExecStart указан только путь к скрипту, но в первой строке server.py есть shebang
#!/usr/bin/env python3, поэтому ядро запускает интерпретатор python3 и передаёт ему скрипт как аргумент. Это подтверждает, что запущен файл из нового расположения app/.

**/proc/14862/fd/** — открытые файловые дескрипторы процесса:
- 0 (stdin) - /dev/null — systemd не подключает сервису ввод
- 1 и 2 (stdout и stderr) - один и тот же сокет socket:[123936] — systemd направляет вывод сервиса в журнал, поэтому сообщения приложения видны в journalctl -u homework-app
- 3 - socket:[123938] — слушающий TCP-сокет порта 8080. Это подтверждает ss -lntpe:  у строки 0.0.0.0:8080  указаны pid=14862, fd=3 и тот же номер ino:123938.

### Маршрут до 1.1.1.1
```bash
ip route get 1.1.1.1
```
1.1.1.1 via 10.0.2.2 dev enp0s3 src 10.0.2.15 uid 100 
Трафик до 1.1.1.1 пойдёт через интерфейс enp0s3 на шлюз 10.0.2.2 — это маршрут по умолчанию из ip route. Реальное подключение к интернету для этого не нужно: команда только показывает, какой маршрут выбрало бы ядро.

### Расхождение в задании
setup.sh устанавливает приложение в /opt/linux-devops-homework/server.py,
а check.sh ожидает /opt/linux-devops-homework/app/server.py (проблема 4).


## Итоговый ход диагностики

1. Запуск setup.sh не удался из-за отсутствия права x у скрипта — запустила через bash
2. systemctl status показал status=203/EXEC: systemd не мог запустить программу.
ls  показал, что файла  app.py  из ExecStart нет, есть  server.py   исправила ExecStart.
3. Программа стала запускаться, но падала с status=1/FAILURE. В journalctl нашлась PermissionError на запись в /var/lib/linux-devops-homework. systemctl show показал, что сервис работает от homework, а `ls -ld` — что каталог принадлежит root с правами 0700 chown homework:homework и chmod 0750.
4. Сервис стал active, но check.sh сообщил, что порт доступен только через loopback.
ss -lntp показал 127.0.0.1:8080, источник host = 127.0.0.1 в app.conf заменила на 0.0.0.0
5. Оставался FAIL по ExecStart: сравнение check.sh и setup.sh показало, что они ожидают разные пути к приложению, создала каталог app, перенесла туда server.py и обновила ExecStart.

Каждая проблема становилась видна только после исправления предыдущей: ошибку доступа к каталогу нельзя было увидеть, пока программа вообще не запускалась, а проблему с адресом, пока сервис падал. После каждого изменения проверяла результат через systemctl status,journalctl, ss и check.sh.


## Итоговая проверка

Скрипт запускался через bash, так как у check.sh нет права на выполнение:
```bash
sudo bash ./scripts/check.sh
```


Linux DevOps Homework Checker

[PASS] systemd ExecStart points to the application
[PASS] service runs as homework:homework
[PASS] state directory ownership and permissions are correct
[PASS] service is enabled
[PASS] service is active
[PASS] running process has the expected UID
[PASS] TCP/8080 listens on 0.0.0.0
[PASS] GET /health returns the expected response
[PASS] fixed systemd unit is saved in the repository
[PASS] fixed application configuration is saved in the repository

Result: 10 passed, 0 failed

