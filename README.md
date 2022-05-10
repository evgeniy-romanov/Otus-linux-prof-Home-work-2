# Дисковая подсистема
## Домашнее задание

 * добавить в Vagrantfile еще дисков;
 * сломать/починить raid;
 * собрать R0/R5/R10 на выбор;
 * прописать собранный рейд в конф, чтобы рейд собирался при загрузке;
 * создать GPT раздел и 5 партиций. В качестве проверки принимаются - измененный Vagrantfile, скрипт для  создания рейда, конф для автосборки рейда при загрузке.
 * доп. задание - Vagrantfile, который сразу собирает систему с подключенным рейдом и смонтированными разделами. После перезагрузки стенда разделы должны автоматически примонтироваться.

## Выполнение задания 

 1 . Клонирую Vagrantfile из репозитория https://github.com/erlong15/otus-linux и добавляю в vagrantfile создание 5го диска.

 2 . Введем команду lsblk для просмотра устройств после создания виртуальной машины.
[vagrant@otuslinux ~]$ sudo lsblk

NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   40G  0 disk
└─sda1   8:1    0   40G  0 part /
sdb      8:16   0  250M  0 disk
sdc      8:32   0  250M  0 disk
sdd      8:48   0  250M  0 disk
sde      8:64   0  250M  0 disk
sdf      8:80   0  250M  0 disk

 3 . Сначала занулим суперблоки на дисках, которые мы будем использовать для построения RAID
[vagrant@otuslinux ~]$ sudo mdadm --zero-superblock --force /dev/sd{b,c,d,e,f}

mdadm: Unrecognised md component device - /dev/sdb
mdadm: Unrecognised md component device - /dev/sdc
mdadm: Unrecognised md component device - /dev/sdd
mdadm: Unrecognised md component device - /dev/sde
mdadm: Unrecognised md component device - /dev/sdf

 4 . Создаем рейд 6
[vagrant@otuslinux ~]$ sudo mdadm --create --verbose /dev/md0 -l 6 -n 5 /dev/sd{b,c,d,e,f}

mdadm: layout defaults to left-symmetric
mdadm: layout defaults to left-symmetric
mdadm: chunk size defaults to 512K
mdadm: size set to 253952K
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.

 5 . Проверим работу рейдов
[vagrant@otuslinux ~]$ sudo cat /proc/mdstat

Personalities : [raid6] [raid5] [raid4]
md0 : active raid6 sdf[4] sde[3] sdd[2] sdc[1] sdb[0]
      761856 blocks super 1.2 level 6, 512k chunk, algorithm 2 [5/5] [UUUUU]

 6 . Подробную информацию о конкретном массиве можно посмотреть командой:

[vagrant@otuslinux ~]$ sudo mdadm -D /dev/md0

/dev/md0:
           Version : 1.2
     Creation Time : Tue May 10 12:09:07 2022
        Raid Level : raid6
        Array Size : 761856 (744.00 MiB 780.14 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 5
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Tue May 10 12:09:31 2022
             State : clean
    Active Devices : 5
   Working Devices : 5
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : 53ed2a11:b817e5fc:1b71f485:d6272403
            Events : 17

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       3       8       64        3      active sync   /dev/sde
       4       8       80        4      active sync   /dev/sdf

где /dev/md0 — имя RAID устройства

 7 . Создадим файла mdadm.conf, в котором находится информация о RAID-массивах и компонентах, которые в них входят.
 
 vagrant@$ sudo su

vagrant@# mkdir /etc/mdadm

vagrant@# echo "DEVICE partitions" > /etc/mdadm/mdadm.conf

vagrant@# mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf

vagrant@$ cat /etc/mdadm/mdadm.conf
DEVICE partitions
ARRAY /dev/md0 level=raid6 num-devices=5 metadata=1.2 name=otuslinux:0 UUID=53ed2a11:b817e5fc:1b71f485:d6272403

 8 . Сломаем массив sdc и проверим статус

[root@otuslinux vagrant]# mdadm /dev/md0 --fail /dev/sdc
mdadm: set /dev/sdc faulty in /dev/md0

[vagrant@otuslinux ~]$ cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md0 : active raid6 sdf[4] sde[3] sdd[2] sdc[1](F) sdb[0]
      761856 blocks super 1.2 level 6, 512k chunk, algorithm 2 [5/4] [U_UUU]
 
 9 . Починим Raid, удалив, а потом добавив блок sdc в md0
[vagrant@otuslinux ~]$ sudo  mdadm --remove /dev/md0 /dev/sdc
mdadm: hot removed /dev/sdc from /dev/md0

[vagrant@otuslinux ~]$ sudo mdadm --add /dev/md0 /dev/sdc
mdadm: added /dev/sdc

 10 .После того как перестроение массива завершилось, размонтируем /dev/md0 и создадим на нем gpt раздел, в разделе четыре партиции, создадим на них файловую систему, смотрю вывод lsblk и монтирую их по каталогам
[vagrant@otuslinux ~]$ umount /dev/md0

[vagrant@otuslinux ~]$ parted -s /dev/md0 mklabel gpt

[vagrant@otuslinux ~]$ /dev/md0 mkpart primary ext4 0% 25%

[vagrant@otuslinux ~]$ /dev/md0 mkpart primary ext4 25% 50%

[vagrant@otuslinux ~]$ /dev/md0 mkpart primary ext4 50% 75%

[vagrant@otuslinux ~]$ /dev/md0 mkpart primary ext4 75% 100%

 11 . Добавлим следующие строки в Vagrantfile в секцию box.vm.provision, делаем vagrant reload --provision, проверяем
mdadm --zero-superblock --force /dev/sd{b,c,d,e,f}

mdadm --create --verbose /dev/md0 -l 6 -n 5 /dev/sd{b,c,d,e,f}

mkdir /etc/mdadm

echo "DEVICE partitions" > /etc/mdadm/mdadm.conf

mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf

parted -s /dev/md0 mklabel gpt

parted /dev/md0 mkpart primary ext4 0% 25%

parted /dev/md0 mkpart primary ext4 25% 50%

parted /dev/md0 mkpart primary ext4 50% 75%

parted /dev/md0 mkpart primary ext4 75% 100%

for i in $(seq 1 4); do sudo mkfs.ext4 /dev/md0p$i; done

mkdir -p /raid/part{1,2,3,4}

for i in $(seq 1 4); do mount /dev/md0p$i /raid/part$i; done

[vagrant@otuslinux ~]$ cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4]
md0 : active raid6 sdc[5] sde[3] sdb[0] sdd[2] sdf[4]
      761856 blocks super 1.2 level 6, 512k chunk, algorithm 2 [5/5] [UUUUU]

[vagrant@otuslinux ~]$ lsblk
NAME      MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda         8:0    0    40G  0 disk
└─sda1      8:1    0    40G  0 part  /

sdb         8:16   0   250M  0 disk
└─md0       9:0    0   744M  0 raid6
 
  ├─md0p1 259:0    0 184.5M  0 md    /raid/part1
  
  ├─md0p2 259:1    0   186M  0 md    /raid/part2
  
  ├─md0p3 259:2    0   186M  0 md    /raid/part3
  
  └─md0p4 259:3    0 184.5M  0 md    /raid/part4

sdc         8:32   0   250M  0 disk
└─md0       9:0    0   744M  0 raid6
  
  ├─md0p1 259:0    0 184.5M  0 md    /raid/part1
  
  ├─md0p2 259:1    0   186M  0 md    /raid/part2
  
  ├─md0p3 259:2    0   186M  0 md    /raid/part3
  
  └─md0p4 259:3    0 184.5M  0 md    /raid/part4

sdd         8:48   0   250M  0 disk
└─md0       9:0    0   744M  0 raid6
  
  ├─md0p1 259:0    0 184.5M  0 md    /raid/part1
  
  ├─md0p2 259:1    0   186M  0 md    /raid/part2
  
  ├─md0p3 259:2    0   186M  0 md    /raid/part3
  
  └─md0p4 259:3    0 184.5M  0 md    /raid/part4

sde         8:64   0   250M  0 disk
└─md0       9:0    0   744M  0 raid6
 
  ├─md0p1 259:0    0 184.5M  0 md    /raid/part1
  
  ├─md0p2 259:1    0   186M  0 md    /raid/part2
    
  ├─md0p3 259:2    0   186M  0 md    /raid/part3
  
  └─md0p4 259:3    0 184.5M  0 md    /raid/part4

sdf         8:80   0   250M  0 disk
└─md0       9:0    0   744M  0 raid6
  
  ├─md0p1 259:0    0 184.5M  0 md    /raid/part1
 
  ├─md0p2 259:1    0   186M  0 md    /raid/part2
  
  ├─md0p3 259:2    0   186M  0 md    /raid/part3
 
  └─md0p4 259:3    0 184.5M  0 md    /raid/part4

## Итог
* Домашнее задание выполнено


