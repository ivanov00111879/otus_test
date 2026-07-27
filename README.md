NFS, FUSE
Что нужно сделать?

запустить 2 виртуальных машины (сервер NFS и клиента);
на сервере NFS должна быть подготовлена и экспортирована директория;
в экспортированной директории должна быть поддиректория с именем upload с правами на запись в неё;
экспортированная директория должна автоматически монтироваться на клиенте при старте виртуальной машины (systemd, autofs или fstab — любым способом);
монтирование и работа NFS на клиенте должна быть организована с использованием NFSv3.

ВМ backend-01 - сервер NFS - 192.168.0.115
ВМ backend-02 - клиент NFS - 192.168.0.121

Настраиваем сервер NFS
Установим сервер NFS:
root@backend-01:~# apt install nfs-kernel-server

Создаём и настраиваем директорию, которая будет экспортирована в будущем
root@backend-01:~# mkdir -p /srv/share/upload
root@backend-01:~# chown -R nobody:nogroup /srv/share
root@backend-01:~# chmod 0777 /srv/share/upload

Cоздаём в файле /etc/exports структуру, которая позволит экспортировать ранее созданную директорию:
root@backend-01:~# cat << EOF > /etc/exports
/srv/share 192.168.0.121/32(rw,sync,root_squash)
EOF

Экспортируем ранее созданную директорию:
root@backend-01:~# exportfs -r

Проверяем экспортированную директорию следующей командой
root@backend-01:~# exportfs -s
/srv/share  192.168.0.121/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)

Настраиваем клиент NFS
Установим пакет с NFS-клиентом
root@backend-02:~# sudo apt install nfs-common

Добавляем в /etc/fstab строку
root@backend-02:~# echo "192.168.0.115:/srv/share/ /mnt nfs vers=3,noauto,x-systemd.automount 0 0" >> /etc/fstab

и выполняем команды:
root@backend-02:~# systemctl daemon-reload
root@backend-02:~# systemctl restart remote-fs.target
root@backend-02:~# mount | grep mnt
systemd-1 on /mnt type autofs (rw,relatime,fd=53,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=9907)

Проверка работоспособности
Заходим на сервер. 
Заходим в каталог /srv/share/upload.
Создаём тестовый файл touch check_file
root@backend-01:~# cd /srv/share/upload/
root@backend-01:/srv/share/upload# touch check_file
root@backend-01:/srv/share/upload# ls
check_file

Заходим на клиент.
Заходим в каталог /mnt/upload. 
Проверяем наличие ранее созданного файла.
root@backend-02:~# cd /mnt/upload/
root@backend-02:/mnt/upload# ls
check_file





ZFS
Что нужно сделать?

Определить алгоритм с наилучшим сжатием:
 определить, какие алгоритмы сжатия поддерживает zfs (gzip, zle, lzjb, lz4);
 создать 4 файловых системы, на каждой применить свой алгоритм сжатия;
 для сжатия использовать либо текстовый файл, либо группу файлов.
Определить настройки пула.
 С помощью команды zfs import собрать pool ZFS.
 Командами zfs определить настройки:
  размер хранилища;
  тип pool;
  значение recordsize;
  какое сжатие используется;
  какая контрольная сумма используется.
Работа со снапшотами:
 скопировать файл из удаленной директории;
 восстановить файл локально. zfs receive;
 найти зашифрованное сообщение в файле secret_message.


Определение алгоритма с наилучшим сжатием

Смотрим список всех дисков, которые есть в виртуальной машине:
ubuntu@ubuntu-22:~$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda      8:0    0   25G  0 disk
├─sda1   8:1    0    1M  0 part
└─sda2   8:2    0   25G  0 part /
sdb      8:16   0  512M  0 disk
sdc      8:32   0  512M  0 disk
sdd      8:48   0  512M  0 disk
sde      8:64   0  512M  0 disk
sdf      8:80   0  512M  0 disk
sdg      8:96   0  512M  0 disk
sdh      8:112  0  512M  0 disk
sdi      8:128  0  512M  0 disk

Установим пакет утилит для ZFS:
ubuntu@ubuntu-22:~$ sudo apt install zfsutils-linux

Создаём пулы из двух дисков в режиме RAID 1:
ubuntu@ubuntu-22:~$ sudo zpool create otus1 mirror /dev/sdb /dev/sdc
ubuntu@ubuntu-22:~$ sudo zpool create otus2 mirror /dev/sdd /dev/sde
ubuntu@ubuntu-22:~$ sudo zpool create otus3 mirror /dev/sdf /dev/sdg
ubuntu@ubuntu-22:~$ sudo zpool create otus4 mirror /dev/sdh /dev/sdi

Смотрим информацию о пулах: zpool list
ubuntu@ubuntu-22:~$ sudo zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
otus1   480M   100K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus2   480M   105K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus3   480M   105K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus4   480M   105K   480M        -         -     0%     0%  1.00x    ONLINE  -

Добавим разные алгоритмы сжатия в каждую файловую систему:
Алгоритм lzjb: zfs set compression=lzjb otus1
Алгоритм lz4:  zfs set compression=lz4 otus2
Алгоритм gzip: zfs set compression=gzip-9 otus3
Алгоритм zle:  zfs set compression=zle otus4

Проверим, что все файловые системы имеют разные методы сжатия:
ubuntu@ubuntu-22:~$ sudo zfs get all | grep compression
otus1  compression           lzjb                   local
otus2  compression           lz4                    local
otus3  compression           gzip-9                 local
otus4  compression           zle                    local

Скачаем /var/log/* во все пулы:
ubuntu@ubuntu-22:~$ sudo cp -r /var/log/* /otus1
ubuntu@ubuntu-22:~$ sudo cp -r /var/log/* /otus2
ubuntu@ubuntu-22:~$ sudo cp -r /var/log/* /otus3
ubuntu@ubuntu-22:~$ sudo cp -r /var/log/* /otus4

Проверим, что /var/log/* был скачан во все пулы:
ubuntu@ubuntu-22:~$ ls -l /otus*
/otus1:
total 1658
/otus2:
total 1376
/otus3:
total 1117
/otus4:
total 4044

Уже на этом этапе видно, что самый оптимальный метод сжатия у нас используется в пуле otus3.
Проверим, сколько места занимает один и тот же файл в разных пулах и проверим степень сжатия файлов:
ubuntu@ubuntu-22:~$ sudo zfs list
NAME    USED  AVAIL     REFER  MOUNTPOINT
otus1  13.8M   338M     13.6M  /otus1
otus2  8.55M   343M     8.38M  /otus2
otus3  6.15M   346M     6.01M  /otus3
otus4  21.6M   330M     21.4M  /otus4

ubuntu@ubuntu-22:~$ sudo zfs get all | grep compressratio | grep -v ref
otus1  compressratio         5.70x                  -
otus2  compressratio         9.28x                  -
otus3  compressratio         12.96x                 -
otus4  compressratio         3.63x                  -

Таким образом, у нас получается, что алгоритм gzip-9 самый эффективный по сжатию.

Определение настроек пула

Скачиваем архив и распаковываем:
ubuntu@ubuntu-22:~$ sudo tar -xzvf zfs_task1.tar.gz
zpoolexport/
zpoolexport/filea

Проверим, возможно ли импортировать данный каталог в пул:
ubuntu@ubuntu-22:~$ sudo zpool import -d zpoolexport/
   pool: otus
     id: 6554193320433390805
  state: ONLINE
status: Some supported features are not enabled on the pool.
        (Note that they may be intentionally disabled if the
        'compatibility' property is set.)
 action: The pool can be imported using its name or numeric identifier, though
        some features will not be available without an explicit 'zpool upgrade'.
 config:

        otus                                ONLINE
          mirror-0                          ONLINE
            /home/ubuntu/zpoolexport/filea  ONLINE
            /home/ubuntu/zpoolexport/fileb  ONLINE

Сделаем импорт данного пула к нам в ОС:
ubuntu@ubuntu-22:~$ sudo zpool import -d zpoolexport/ otus
ubuntu@ubuntu-22:~$ sudo zpool status
  pool: otus
 state: ONLINE
status: Some supported and requested features are not enabled on the pool.
        The pool can still be used, but some features are unavailable.
action: Enable all features using 'zpool upgrade'. Once this is done,
        the pool may no longer be accessible by software that does not support
        the features. See zpool-features(7) for details.

Уточняем параметры импортированного пула:
ubuntu@ubuntu-22:~$ sudo zfs get available otus
NAME  PROPERTY   VALUE  SOURCE
otus  available  350M   -
ubuntu@ubuntu-22:~$ sudo zfs get readonly otus
NAME  PROPERTY  VALUE   SOURCE
otus  readonly  off     default
ubuntu@ubuntu-22:~$ sudo zfs get recordsize otus
NAME  PROPERTY    VALUE    SOURCE
otus  recordsize  128K     local
ubuntu@ubuntu-22:~$ sudo zfs get checksum otus
NAME  PROPERTY  VALUE      SOURCE
otus  checksum  sha256     local

Работа со снапшотами:

Скачаем файл, указанный в задании:
ubuntu@ubuntu-22:~$ wget -O otus_task2.file --no-check-certificate https://drive.usercontent.google.com/download?id=1wgxjih8YZ-cqLqaZVa0lA3h3Y029c3oI&export=download
Восстановим файловую систему из снапшота:
ubuntu@ubuntu-22:~$ sudo zfs receive otus/test@today < otus_task2.file

Далее, ищем в каталоге /otus/test файл с именем “secret_message”:
root@ubuntu-22:~# find /otus/test -name "secret_message"
/otus/test/task1/file_mess/secret_message
Смотрим содержимое найденного файла:
root@ubuntu-22:~# cat /otus/test/task1/file_mess/secret_message
https://otus.ru/lessons/linux-hl/








Обновление ядра системы

Задание
1. Запустите ВМ c Ubuntu.
2. Обновите ядро ОС на новейшую стабильную версию из mainline-репозитория.

Подключаемся к ВМ с ОС Ubuntu и проверяем текущую версию ядра командой uname -r 

<img width="682" height="324" alt="image" src="https://github.com/user-attachments/assets/b8d5f3f4-e9bc-4c77-acb7-0febf37e7f49" />

Далее в браузере переходим в репозиторий https://kernel.ubuntu.com/mainline, находим свежую версию ядра для нашей архитектуры. На текущий момент последняя версия 7.1.2. Переходим в каталог и находим ПО для нужной нам архитектуры: 

<img width="1002" height="405" alt="image" src="https://github.com/user-attachments/assets/0128c5ed-1c18-4a6b-80e6-cbe04e9b9842" />

Возвращаемся в ВМ, создаем каталог командой mkdir kernel && cd kernel и загружаем ПО по ссылкам из репозитория командами:
wget https://kernel.ubuntu.com/mainline/v7.1.2/amd64/linux-headers-7.1.2-070102-generic_7.1.2-070102.202606271039_amd64.deb
wget https://kernel.ubuntu.com/mainline/v7.1.2/amd64/linux-headers-7.1.2-070102_7.1.2-070102.202606271039_all.deb
wget https://kernel.ubuntu.com/mainline/v7.1.2/amd64/linux-image-unsigned-7.1.2-070102-generic_7.1.2-070102.202606271039_amd64.deb
wget https://kernel.ubuntu.com/mainline/v7.1.2/amd64/linux-modules-7.1.2-070102-generic_7.1.2-070102.202606271039_amd64.deb

Устанавливаем все пакеты сразу:
sudo dpkg -i *.deb 

Проверяем, что ядро появилось в /boot.
ls -al /boot

Перезагружаем ВМ и проверяем текущую версию ядра командой uname -r 

<img width="873" height="464" alt="2" src="https://github.com/user-attachments/assets/9716cd60-fad5-4a48-b1e1-301bf2673942" />
