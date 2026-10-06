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

## Дополнительные наблюдения

При необходимости зафиксируйте здесь сведения о процессе, `/proc`, маршрутизации или другие результаты исследования системы, которые не относятся только к одной проблеме.


## Итоговый ход диагностики

Кратко опишите весь путь от исходного нерабочего состояния до исправленного сервиса: в каком порядке проявлялись проблемы и как одна проверка приводила к следующей.


## Итоговая проверка

Вставьте полный вывод успешного запуска:

```bash
sudo ./scripts/check.sh
```

```text
# вывод check.sh
```
