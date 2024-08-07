## Не загружается Debian/Ubuntu/Mint/Kali с initramfs в BusyBox

+ [initbusybox error] https://winitpro.ru/index.php/2019/11/22/initramfs-busybox-v-ubuntu/

`
Initramfs – это начальная файловая система в ОЗУ, основанная на tmpfs, которая не использует отдельное блочное устройство. 
Как и initrd, она содержит утилиты и скрипты, требуемые для монтирования файловых систем перед вызовом init, который располагается на корневой файловой системе.
`

рассмотрим варианты решения проблем, когда виртуальный или физический серверы на базе Ubuntu/Mint/Kali не загружаются и отваливается в busybox в момент инициализации initramfs. При этом Linux не загружается, и пользователю доступна только командная строка initramfs.

### Проблема с суперблоком

Если Ubuntu свалилась в busybox при инициализации initramfs, возможно на диске оказался испорченный суперблок. Linux хранит несколько копий суперблоков.
Для восстановления в случае такой проблемы, нам нужно загрузиться с образа/диска и запустить Terminal. 

После загрузки, в терминале вводим команду:
```bash
sudo fdisk -l|grep Linux|grep -Ev 'swap'
```

Команда вернет информацию о нашем разделе:
```bash
/dev/vda2 4096 83884031 83879936 40G Linux filesystem
```

Запомните имя раздела и укажите его в следующей команде:
```bash
sudo dumpe2fs /dev/vda2 | grep superblock
```

Команда вернет список запасных суперблоков:
```bash
dumpe2fs 1.47.1 (20-May-2024)
  Primary superblock at 0, Group descriptors at 1-14
  Backup superblock at 32768, Group descriptors at 32769-32782
  Backup superblock at 98304, Group descriptors at 98305-98318
  Backup superblock at 163840, Group descriptors at 163841-163854
  Backup superblock at 229376, Group descriptors at 229377-229390
  Backup superblock at 294912, Group descriptors at 294913-294926
  Backup superblock at 819200, Group descriptors at 819201-819214
  Backup superblock at 884736, Group descriptors at 884737-884750
  Backup superblock at 1605632, Group descriptors at 1605633-1605646
  Backup superblock at 2654208, Group descriptors at 2654209-2654222
  Backup superblock at 4096000, Group descriptors at 4096001-4096014
  Backup superblock at 7962624, Group descriptors at 7962625-7962638
  Backup superblock at 11239424, Group descriptors at 11239425-11239438
  Backup superblock at 20480000, Group descriptors at 20480001-20480014
  Backup superblock at 23887872, Group descriptors at 23887873-23887886
```

Мы будем использовать второй резервный суперблок для замены поврежденного (можно выбрать любой, кроме Primary). Выполним проверку диска с использованием резевного суберблока для восстановления:

```bash
sudo fsck -b 98304 /dev/vda2 -y
```
Если вы получите вывод:
```

fsck from util-linux 2.31.1
e2fsck 1.44.1 (24-Mar-2018)
/dev/vda2 is mounted.
e2fsck: Cannot continue, aborting

```
Нужно отмонтировать раздел:
```
umount /dev/vda2
```

После успешного выполнения замены суперблока, вы должны получить такое сообщение:
```

fsck from util-linux 2.31.1
e2fsck 1.44.1 (24-Mar-2018)
/dev/vda2 was not cleanly unmounted, check forced.
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information
Free blocks count wrong for group #231 (32254, counted=32253).
Fix? yes
 Free blocks count wrong for group #352 (32254, counted=32248).
Fix? yes
Free blocks count wrong for group #358 (32254, counted=27774).
Fix? yes
..........
/dev/vda2: ***** FILE SYSTEM WAS MODIFIED *****
/dev/vda2: 85986/905464576 files (0.2% non-contiguous), 3904682/905464576 blocks
```

Теперь перезагрузите компьютеры, отключив диск с дистрибутивом и все должно быть в порядке.