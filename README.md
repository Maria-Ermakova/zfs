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

2. Определение настроек пула

#Скачаем архив в домашнюю директорию, а потом разархивируем его
wget -O archive.tar.gz --no-check-certificate 'https://downloader.disk.yandex.ru/disk/7aef49f8727c3b2ccf3655961f4f4e8aa488084a8514cb88113b11628
f78ec6c/6981deef/32RThlEBa5ILS5KnIMNz6D6CDh_ZwzrzvttzG3Z1mYO156m6RSoEZpzw1wyR_JQQ8n-1oPiYlDJ6UXha6myBAA%3D%3D?uid=82055994&filename=zfs_task1.tar.gz&disposi
tion=attachment&hash=&limit=0&content_type=application%2Fx-gzip&owner_uid=82055994&fsize=7275140&hid=4bbbdf44de8f15e8c068e444377f736f&media_type=compressed&
tknv=v3&etag=157e13606ae3211a54f6c70e9a62cfd1'

tar -xzvf archive.tar.gz
 zpoolexport/
 zpoolexport/filea
 zpoolexport/fileb

#Проверим, какие пулы доступны для импорта из указанного каталога, состояние данных пула:
zpool import -d zpoolexport/
  pool: otus
     id: 6554193320433390805
  state: ONLINE
status: Some supported features are not enabled on the pool.
        (Note that they may be intentionally disabled if the
        'compatibility' property is set.)
 action: The pool can be imported using its name or numeric identifier, though
        some features will not be available without an explicit 'zpool upgrade'.
 config:

        otus                         ONLINE
          mirror-0                   ONLINE
            /root/zpoolexport/filea  ONLINE
            /root/zpoolexport/fileb  ONLINE

#импортируем пул и проверим его статус и состав после импорта
zpool import -d zpoolexport/ otus

zpool status
  pool: otus
 state: ONLINE
status: Some supported and requested features are not enabled on the pool.
        The pool can still be used, but some features are unavailable.
action: Enable all features using 'zpool upgrade'. Once this is done,
        the pool may no longer be accessible by software that does not support
        the features. See zpool-features(7) for details.
config:

        NAME                         STATE     READ WRITE CKSUM
        otus                         ONLINE       0     0     0
          mirror-0                   ONLINE       0     0     0
            /root/zpoolexport/filea  ONLINE       0     0     0
            /root/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors

zpool get all otus
zfs get all otus

#если интересуют конкретные параметры а не весь список, то
zfs get available otus
 NAME  PROPERTY   VALUE  SOURCE
 otus  available  350M   -
zfs get readonly otus
 NAME  PROPERTY  VALUE   SOURCE
 otus  readonly  off     default
zfs get recordsize otus
 NAME  PROPERTY    VALUE    SOURCE
 otus  recordsize  128K     local
zfs get compression otus
 NAME  PROPERTY     VALUE     SOURCE
 otus  compression  zle       local
