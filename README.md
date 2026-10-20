# homework3
Задание предсталено в [task.md](task.md)

# Сборка и запуск
Для сборки необходим cmake и компилятор для c++ с поддержкой c++23
```
mkdir build && cd build
cmake .. 
cmake --build .
```

Для тестирования работоспособности необходим образ ФС. Тестировалось на linux alpine
```
mkdir -p /tmp/alpine
cd /tmp/alpine
wget https://dl-cdn.alpinelinux.org/alpine/v3.19/releases/x86_64/alpine-minirootfs-3.19.0-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.19.0-x86_64.tar.gz
rm alpine-minirootfs-3.19.0-x86_64.tar.gz
```

Пример запуска контейнера
```
./runtime --rootfs /tmp/alpine /bin/ls
```

# Реализация
Работу программы можно разделить на две части:
- родитель подготавливает ресурсы (выделяет память под стек дочерниму процессу), создает дочерний процесс через `clone()`, настраивает UID/GID-маппинг и сообщает дочернему процессу о готовности к работе.
- потомок выполняется в выделенном пространстве, ждет сигнала от родительского процесса о готовности, по получении выполняет переданную через аргументы целевую программу.

Все ошибки по ходу выполнения в дочернем процессе передаются родителю через pipe функцией `die()`

Родительский процесс записывает в `/proc/<pid>/uid_map` и `/proc/<pid>/gid_map` строки вида `"0 <uid> 1"`, отображая root внутри контейнера на текущего пользователя снаружи. Перед записью gid_map в `/proc/<pid>/setgroups` пишется "deny" — требование ядра при непривилегированном маппинге GID. После этого сообщает дочернему процессу о возможности выполнять целевую команду

Дочерний процесс после получения сигнала (байта "G") выполняет:
- устанавливает hostname (для того, чтобы убедиться в команде hostname, что находимся внутри контейнера) через `sethostname`. Работает благодаря флагу `CLONE_NEWUTS`.
- монтирование `/proc` в `<rootfs>/proc` (если в `<rootfs>` не было директории `/proc`, то создаст) и `tmpfs` в  `<rootfs/dev>` (изолирует хостовые /dev/sda и тд). Возможно благодаря флагу  `CLONE_NEWPID` и уже настроенному маппингу UID в родительском процессе
- сменяет корень и рабочий каталог через `chroot` и `chdir`
- запускает целевую команду через `execvp`

# Результат
## /bin/ls
Результат на хосте
```
/bin/ls /
bin                boot   dev  home  lib32  lib.usr-is-merged  media  opt   root  sbin                snap  swap.img  tmp  var
bin.usr-is-merged  cdrom  etc  lib   lib64  lost+found         mnt    proc  run   sbin.usr-is-merged  srv   sys       usr
```
В контейнере
```
./runtime --rootfs /tmp/alpine/ /bin/ls
bin    dev    etc    home   lib    media  mnt    opt    proc   root   run    sbin   srv    sys    tmp    usr    var
```
## /bin/ps
Результат на хосте
```
/bin/ps
    PID TTY          TIME CMD
  53268 pts/2    00:00:00 bash
  59620 pts/2    00:00:00 ps
```

В контейнере
```
./runtime --rootfs /tmp/alpine/ /bin/ps
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/ps
```

## id
Результат на хосте
```
id
uid=1000(alust) gid=1000(alust) groups=1000(alust),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),100(users),114(lpadmin),984(docker)
```

В контейнере
```
./runtime --rootfs /tmp/alpine/ id
uid=0(root) gid=0(root) groups=65534(nobody),65534(nobody),65534(nobody),65534(nobody),65534(nobody),65534(nobody),65534(nobody),65534(nobody),0(root)
```
## hostname
Результат на хосте
```
hostname
alust-VivoBook-ASUSLaptop-X512DA-A512DA
```

В контейнере
```
/runtime --rootfs /tmp/alpine/ hostname
my_runtime
```

#  Монтирование /proc без изоляции mount-пространства
Согласно [man](https://man7.org/linux/man-pages/man7/user_namespaces.7.html) для монтирования /proc необходимо наличие привелегии `CAP_SYS_ADMIN` в PID namespace и mount namespace. Если не изолировать mount-пространство (убрать флаг CLONE_NEWNS), то mount namespace внутри дочернего процесса - родительский, на который нет привелегии `CAP_SYS_ADMIN`, поэтому mount завершается с EPERM, несмотря на то что требование с PID namespace выполнено благодаря CLONE_NEWPID и маппингу UID/GID.