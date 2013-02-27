---
layout: post
title: "Определение ФС на LVS томе"
description: "Как определить тип файловой системе на LVS томе снаружи"
category: howto
tags: [lvs, howto, fs]
---
{% include JB/setup %}
Для определения FS на LVS томе нам необходимо получить метаданные данные с тома, определяющие тип FS.

Для начала нужно посотреть смещение на томе, чтобы определить откуда начинаются необходимые нам данные. Для этого можно выполнить команду:
{% highlight bash %}
$ mdadm -E /dev/sas00/51501_24
/dev/sas00/51501_24:
          Magic : a92b4efc
        Version : 1.2
    Feature Map : 0x1
     Array UUID : a11a9dae:fa6f187e:fdce9334:ec1fe46b
           Name : xen10:md51501_24
  Creation Time : Wed Sep 26 12:01:52 2012
     Raid Level : raid1
   Raid Devices : 2

 Avail Dev Size : 10491904 (5.00 GiB 5.37 GB)
     Array Size : 10491880 (5.00 GiB 5.37 GB)
  Used Dev Size : 10491880 (5.00 GiB 5.37 GB)
    **Data Offset : 2048 sectors**
   Super Offset : 8 sectors
          State : clean
    Device UUID : 58297892:37844649:cb27316e:2e2f8c0e

Internal Bitmap : 8 sectors from superblock
    Update Time : Fri Oct 12 10:30:43 2012
       Checksum : 6b31c0a0 - correct
         Events : 6786


   Device Role : Active device 0
   Array State : AA ('A' == active, '.' == missing)
{% endhighlight %}

Далее мы можем получить суберблок и определить тип файловой системы:
{% highlight bash %}
dd if=/dev/sas00/51501_24 skip=2048 bs=1k count=1024 | file - 
/dev/stdin: Linux rev 1.0 ext4 filesystem data, UUID=afb28ffa-9663-4f2e-94cb-9d05abfd1b76 (needs journal recovery) (extents) (large files) (huge files)
{% endhighlight %}

Том, который использовался для примера содержит FS ext4