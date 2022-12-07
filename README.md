#  Файловые системы и LVM

Задание:

```text
На имеющемся образе centos/7 - v. 1804.2
1) Уменьшить том под / до 8G
2) Выделить том под /home
3) Выделить том под /var - сделать в mirror
4) /home - сделать том для снапшотов
5) Прописать монтирование в fstab. Попробовать с разными опциями и разными файловыми системами ( на выбор)
```

Выполнение:
1. Уменьшить том под / до 8G

Подготовил временный том для / раздела и создал на нём файловую систему:
```bash
 pvcreate /dev/sdb
vgcreate vg_root /dev/sdb
lvcreate -n lv_root -l +100%FREE /dev/vg_root
mkfs.xfs /dev/vg_root/lv_root
mount /dev/vg_root/lv_root /mnt
```
Скопировал все данные с / раздела в /mnt:
```bash
xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
```
Сымитировал текущий root -> сделал в него chroot и обновил grub:
```bash
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
chroot /mnt/
grub2-mkconfig -o /boot/grub2/grub.cfg
```
Обновил образ initrd:
```bash
cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;s/.img//g"` --force; done
```
В файле /boot/grub2/grub.cfg заменил rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root

Перезагружаем ОС

После изменил размер старой VG и вернул на него рут. Для этого удалил старый LV размеров в 40G и создаем новый на 8G:
```bash
lvremove /dev/VolGroup00/LogVol00
lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00

```
Создал файловую систему и скопировал данные:
```bash
mkfs.xfs /dev/VolGroup00/LogVol00
mount /dev/VolGroup00/LogVol00 /mnt
xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt
```
Переконфигурировал grub, за исключением правки /etc/grub2/grub.cfg
```bash
 for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
chroot /mnt/
grub2-mkconfig -o /boot/grub2/grub.cfg
cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;s/.img//g"` --force; done
```
2. Выделить том под /var - сделать в mirror

На свободных дисках создал зеркало:
```bash
vcreate /dev/sdc /dev/sdd
vgcreate vg_var /dev/sdc /dev/sdd
lvcreate -L 950M -m1 -n lv_var vg_var
```
Выделил том под /var в зеркало:
Создал на нем ФС и перестил туда /var:
```bash
mkfs.ext4 /dev/vg_var/lv_var
mount /dev/vg_var/lv_var /mnt
cp -aR /var/* /mnt/ # rsync -avHPSAX /var/ /mnt/
```
На всякий случай сохранил содержимое старого var:
```bash
mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
```
Смонтировал новый var в каталог /var:
```bash
umount /mnt
mount /dev/vg_var/lv_var /var
```
3. Прописать монтирование /var в fstab. 
Исправил fstab для автоматического монтирования /var:
```bash
echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab
```
После чего перезагрузился в новый (уменьшенный root) и удалил
временную Volume Group:
```bash
lvremove /dev/vg_root/lv_root
vgremove /dev/vg_root
pvremove /dev/sdb
```
4. Выделить том под /home
Выделил том под /home по тому же принципу что делали для /var:
```bash
lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
mkfs.xfs /dev/VolGroup00/LogVol_Home
mount /dev/VolGroup00/LogVol_Home /mnt/
cp -aR /home/* /mnt/
rm -rf /home/*
umount /mnt
mount /dev/VolGroup00/LogVol_Home /home/
```
5. Прописать монтирование /home в fstab. 
Исправил fstab для автоматического монтирования /home
```bash
echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab
```
Сгенерировал файлы в /home/:
```bash
touch /home/file{1..20}
```
5. /home - сделать том для снапшотов
Снял снапшот:
```bash
lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
```
Удалил часть файлов:
```bash
rm -f /home/file{11..20}
```
Процесс восстановления со снапшота:
```bash
umount /home
lvconvert --merge /dev/VolGroup00/home_snap
mount /home
```
Окончательный вывод lsblk:
```text
[vagrant@lvm ~]$ lsblk
NAME                       MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                          8:0    0   40G  0 disk 
├─sda1                       8:1    0    1M  0 part 
├─sda2                       8:2    0    1G  0 part /boot
└─sda3                       8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00    253:0    0    8G  0 lvm  /
  ├─VolGroup00-LogVol01    253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol_Home 253:5    0    2G  0 lvm  /home
sdb                          8:16   0   10G  0 disk 
sdc                          8:32   0    2G  0 disk 
├─vg_var-lv_var_rmeta_0    253:2    0    4M  0 lvm  
│ └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0   253:3    0  952M  0 lvm  
  └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
sdd                          8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1    253:4    0    4M  0 lvm  
│ └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1   253:6    0  952M  0 lvm  
  └─vg_var-lv_var          253:7    0  952M  0 lvm  /var
sde                          8:64   0    1G  0 disk 

```



