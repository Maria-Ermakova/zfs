# zfs
Практические навыки работы с ZFS

1. Определение алгоритма с наилучшим сжатием

#используемые полезные команды
zpool status - показывает информацию о каждом диске, состоянии сканирования и об ошибках чтения, записи и совпадения хэш-сумм
zpool list - показывает информацию о размере пула, количеству занятого и свободного места, дедупликации и т.д.

#дано: ВМ ubuntu 24.04 server + 8 дисков по 512М
su -

#Установим пакет утилит для ZFS:
apt install zfsutils-linux

#Создаём 4 пула, каждый из двух дисков в режиме RAID 1:
zpool create otus1 mirror /dev/sdb /dev/sdc
zpool create otus2 mirror /dev/sdd /dev/sde
zpool create otus3 mirror /dev/sdf /dev/sdg
zpool create otus4 mirror /dev/sdh /dev/sdi

#Добавим разные алгоритмы сжатия ( gzip, zle, lzjb, lz4) в каждую файловую систему. Сжатие файлов будет работать только с файлами, которые были добавлены после включение настройки сжатия:
zfs set compression=lzjb otus1
zfs set compression=lz4 otus2
zfs set compression=gzip-9 otus3
zfs set compression=zle otus4

#Проверим, что все файловые системы имеют разные методы сжатия:
zfs get all | grep compression

#Скопируем одну и ту же папку с логами во все пулы: 
for i in {1..4}; do cp -r /var/log/* /otus$i; done
ls -l /otus*
#Уже на этом этапе видно, что самый оптимальный метод сжатия у нас используется в пуле otus3.Проверим, сколько места занимает папка в разных пулах и степень сжатия файлов:
zfs list
NAME    USED  AVAIL  REFER  MOUNTPOINT
otus1  4.01M   348M  3.89M  /otus1
otus2  2.42M   350M  2.29M  /otus2
otus3  1.66M   350M  1.53M  /otus3
otus4  5.63M   346M  5.50M  /otus4

zfs get all | grep compressratio | grep -v ref
otus1  compressratio         11.74x                 -
otus2  compressratio         19.92x                 -
otus3  compressratio         29.93x                 -
otus4  compressratio         8.28x                  -

#Вывод: алгоритм gzip-9 самый эффективный по сжатию

