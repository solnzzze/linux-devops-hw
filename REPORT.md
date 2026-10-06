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
