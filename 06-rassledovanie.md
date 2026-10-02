# Задание 6. Охота на вредоносный процесс

## Шаг 1. Имитация заражения.

1. `/tmp/kworker 3000` -
```
[1] 2143
```

`rm /tmp/kworker`

2. `python3 -m http.server 8080 > /dev/null 2>&1 &` -
```
[2] 2145
```

(Что вы сделали:
скопировали безобидную программу sleep под видом системного процесса kworker ;
запустили её из временного каталога /tmp ;
удалили файл с диска — процесс продолжает работать;
открыли на машине сетевой порт 8080, доступный снаружи.)

## Шаг 2. Расследование.

3. `ps -eo pid,user,cmd | grep kworker` -
```
   1664 root     [kworker/0:0H-kblockd]
   2118 root     [kworker/0:1-events]
   2119 root     [kworker/0:2-events]
   2123 root     [kworker/0:0-mpt_poll_0]
   2143 user     /tmp/kworker 3000
   2149 user     grep kworker
```

4. `sudo ls -l /proc/*/exe 2>/dev/null | grep -E '/tmp|/dev/shm|deleted'` -
```
[sudo] пароль для user: 
lrwxrwxrwx 1 user             user             0 окт  3 03:17 /proc/2143/exe -> /tmp/kworker (deleted)
```

5. `sudo ss -tulpn | grep 8080` - 
```
tcp   LISTEN 0      5                               0.0.0.0:8080      0.0.0.0:*    users:(("python3",pid=2145,fd=3))
```

6. `sudo lsof -i :8080` - 
```
COMMAND  PID USER FD   TYPE DEVICE SIZE/OFF NODE NAME
python3 2145 user 3u  IPv4  35304      0t0  TCP *:http-alt (LISTEN)
```
7. `ps -o pid,ppid,user,cmd -p 2145` -
```
    PID    PPID USER     CMD
   2145    1072 user     python3 -m http.server 8080
```

## Шаг 3. Устранение.

8. `kill 2145 && kill 1072` - 
```
[2]+  Завершено      python3 -m http.server 8080 > /dev/null 2>&1
```

## Итоговый отчёт

1. 

# Задание 6. Охота на вредоносный процесс

## Шаг 1. Имитация заражения.

1. `/tmp/kworker 3000` -
```
[1] 2143
```

`rm /tmp/kworker`

2. `python3 -m http.server 8080 > /dev/null 2>&1 &` -
```
[2] 2145
```

(Что вы сделали:
скопировали безобидную программу sleep под видом системного процесса kworker ;
запустили её из временного каталога /tmp ;
удалили файл с диска — процесс продолжает работать;
открыли на машине сетевой порт 8080, доступный снаружи.)

## Шаг 2. Расследование.

3. `ps -eo pid,user,cmd | grep kworker` -
```
   1664 root     [kworker/0:0H-kblockd]
   2118 root     [kworker/0:1-events]
   2119 root     [kworker/0:2-events]
   2123 root     [kworker/0:0-mpt_poll_0]
   2143 user     /tmp/kworker 3000
   2149 user     grep kworker
```

4. `sudo ls -l /proc/*/exe 2>/dev/null | grep -E '/tmp|/dev/shm|deleted'` -
```
[sudo] пароль для user: 
lrwxrwxrwx 1 user             user             0 окт  3 03:17 /proc/2143/exe -> /tmp/kworker (deleted)
```

5. `sudo ss -tulpn | grep 8080` - 
```
tcp   LISTEN 0      5                               0.0.0.0:8080      0.0.0.0:*    users:(("python3",pid=2145,fd=3))
```

6. `sudo lsof -i :8080` - 
```
COMMAND  PID USER FD   TYPE DEVICE SIZE/OFF NODE NAME
python3 2145 user 3u  IPv4  35304      0t0  TCP *:http-alt (LISTEN)
```
7. `ps -o pid,ppid,user,cmd -p 2145` -
```
    PID    PPID USER     CMD
   2145    1072 user     python3 -m http.server 8080
```

## Шаг 3. Устранение.

8. `kill 2145 && kill 1072` - 
```
[2]+  Завершено      python3 -m http.server 8080 > /dev/null 2>&1
```


## Итоговый отчет

1. **Что обнаружено: какие процессы, их PID, владелец, родитель**

|PID|Пользователь|Родитель|Команда|
|---|---|---|---|
|2143|user|----|`/tmp/kworker 3000`|
|2145|user|1072|`python3 -m http.server 8080`|

> Процесс 2143 был запущен из `/tmp` под именем `kworker`, похожим на имя системного процесса Linux.

> Процесс 2145 запустил простой HTTP-сервер Python на порту 8080. Его родителем был процесс Bash с PID 1072.

2. **По каким признакам: какой командой найден каждый из четырёх признаков, вывод команд;**

- Подзрительные процессы `kworker`

Команда `ps -eo pid,user,cmd | grep kworker` :
```
1664 root  [kworker/0:0H-kblockd]
2118 root  [kworker/0:1-events]
2119 root  [kworker/0:2-events]
2123 root  [kworker/0:0-mpt_poll_0]
2143 user  /tmp/kworker 3000
2149 user  grep kworker
```
Процесс 2143 подозрителен, потому что называется kworker, но запускается от пользователя user и находится в `/tmp`. Настоящие kworker являются системными kernel-процессами и имеют другой вид.
- Сам исполняемый файл был удалён

Команда `sudo ls -l /proc/*/exe 2>/dev/null | grep -E '/tmp|/dev/shm|deleted'` :
```
lrwxrwxrwx 1 user user 0 окт 3 03:17 /proc/2143/exe -> /tmp/kworker (deleted)
```
Это показывает, что процесс 2143 продолжает работать, хотя файл `/tmp/kworker` уже удалён с диска.

- Открытый порт

Команда `sudo ss -tulpn | grep 8080` :
```
tcp LISTEN 0 5 0.0.0.0:8080 0.0.0.0:* users:(("python3",pid=2145,fd=3))
```
Процесс 2145 открыл TCP-порт 8080 и принимает соединения на всех сетевых интерфейсах (0.0.0.0). Это подозрительно, поскольку работающая программа может продолжать использовать удалённый файл до завершения процесса.

- определение владельца и родителя

Команда: `ps -o pid,ppid,user,cmd -p 2145` :
```
    PID    PPID USER     CMD
   2145   1072 user     python3 -m http.server 8080
```

Из вывода видно, что сервер запущен пользователем user, а его родителем является Bash с PID 1072. Это позволяет установить происхождение процесса и понять, какой процесс его запустил.

3. **Почему это подозрительно: объясните каждый признак;**

Основным подозрительным процессом является `2143`.

- Он называется kworker, т.e. маскируется под системный процесс.
- Он запущен от обычного пользователя user.
- Исполняемый файл находится в `/tmp`.
- Файл `/tmp/kworker` был удалён, но процесс продолжал работать.
- Его PID можно было найти через `/proc`, где настоящий исполняемый файл отображался как `/tmp/kworker (deleted)`.

4. **Как устранено: какие действия выполнены и как проверено, что угроза снята;**

- Для остановки HTTP-сервера была выполнена команда `kill 2145`.

После этого сервер Python завершился.
- Затем остановка еще одного процесса `kill 1072`.

Она завершила родительский процесс Bash, после чего в терминале появилось:
```
[2]+  Завершено      python3 -m http.server 8080 > /dev/null 2>&1
```

Для окончательной проверки использовалось:
```
ps -eo pid,user,cmd | grep kworker
sudo ss -tulpn | grep 8080
sudo ls -l /proc/*/exe 2>/dev/null | grep -E '/tmp|/dev/shm|deleted'
```
Если подозрительный PID и порт 8080 больше не отображаются, соответствующие процессы завершены.


5. **Вывод: почему при поиске вредоносного процесса нельзя доверять его имени, а `/proc/PID/exe` — можно.**
> При поиске подозрительного процесса нельзя доверять только его имени, потому что обычная программа может назвать себя как угодно. Эта ссылка показывает фактический исполняемый файл, связанный с процессом.
```
/proc/2143/exe -> /tmp/kworker (deleted)
```