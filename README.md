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