# Решение входа в систему без пароля

Примечание - данный вариант протестирован на Centos 7. На других системах по рассказам гугла некоторые моменты могут отличаться.

## Вариант 1 входа без пароля 

1. При запуске, в момент выбора системы нажимаем 'e', попадаем в редактор параметров загрузки

2. Находим в параметрах строку, начинающуюся с 'linux16'. В данной строке оставляем название загружаемого ядра и UUID рутового диска. Из остальных параметров в данной строке можно оставить только важные, а всю прочую шваль убрать (иначе загрузка системы выполняется некоректно или виснет). Изменяем значение ro на rw для того, чтобы файловая система была доступна не только на чтение, но и на запись. 

3. В конце строки добавляем параметр single init=/bin/bash (можно ввести и init=/bin/sh, результат будет тем же). Для выхода и продолжения загрузки нажимаем ctrl-x. В определенный момент загрузка системы завершится с появлением строки bash-4.2# и можно вводить команды, например смены пароля.

4. Есть одна тонкость - необходимо проверить параметр SELINUX /etc/selinux/config (в новых системах). Нужно заменить строку SELINUX=enforcing на SELINUX=disabled. В противном случае после смены пароля и перезагрузки получим неприятную ситуацию - при вводе логина и пароля будет сообщение Login incorrect

	- Если SELINUX еще не выключен и в параметрах загрузки оставлен параметр 'ro' (read-only), то необходимо ввести следующие команды:  
		mount -o remount,rw /  
		rpm --setperms `rpm -qa`  
		passwd  
		Enter new password: (вводим новый пароль)  
		Repeat... (повторяем новый пароль)  
		vi /etc/selinux/config (меняем SELINUX=enforcing на SELINUX=disabled)  
		sync  
		mount -o remount,ro /  

	- Если SELINUX  уже выключен ранее и в параметрах загрузки параметр 'ro' заменен на 'rw', то для смены пароля достаточно одной команды:  
		passwd  
		Enter new password: (вводим новый пароль)  
		Repeat... (повторяем новый пароль)  

5. Если выйти командой exit, то система подвисает. Причину не нашел. Гугль говорит следующее: "... и затем отключите питание компьютера/перезагрузите физической кнопкой." (https://hackware.ru/?p=3801)

## Вариант 2 входа без пароля

1. Выполняем п.1 и п.2 варианта 1

2. В конце строки добавляем параметр rd.break. Для выхода и продолжения загрузки нажимаем ctrl-x. В определенный момент загрузка системы завершится с появлением строки switch_root:/#. Необходимо ввести следующую команду:  
		chroot /sysroot  

3. Появится строка sh-4.2# и можно вводить команды для смены пароля:  
		passwd  
		Enter new password: (вводим новый пароль)  
		Repeat... (повторяем новый пароль)  

4. Выполняем перезагрузку  
		exit  
		reboot  

## Вариант 3 входа без пароля

1. Выполняем п.1 и п.2 варианта 1

2. В конце строки добавляем параметр init=/sysroot/bin/sh. Для выхода и продолжения загрузки нажимаем ctrl-x. В определенный момент загрузка системы завершится с появлением строки :/#. Необходимо ввести следующую команду:  
		chroot /sysroot  

3. строка останется такой же :/#, но уже можно вводить команду для смены пароля:  
		passwd  
		Enter new password: (вводим новый пароль)  
		Repeat... (повторяем новый пароль)  

4. Выполняем перезагрузку  
		exit  
		reboot  


# Решение переименовывания Volume Group в LVM

1. Разворачиваем виртуалку с ДЗ-3 (Файловые системы и LVM)

2. Отключаем режим SELINUX и перезагружаем  
		vi /etc/selinux/config (меняем SELINUX=enforcing на SELINUX=disabled)  

3.  Посмотрим текущее значение Volume Group  
		[root@lvm vagrant]# vgs  
	VG         #PV #LV #SN Attr   VSize   VFree  
	VolGroup00   1   2   0 wz--n- <38.97g    0  

4. Переименуем Volume Group  
		[root@lvm vagrant]# vgrename VolGroup00 OtusRoot  
	Volume group "VolGroup00" successfully renamed to "OtusRoot"  

5. Изменим значение VolGroup00 на OtusRoot в файле /etc/fstab  
		[root@lvm vagrant]# vi /etc/fstab  
		[root@lvm vagrant]# cat  /etc/fstab  
	/etc/fstab  
	Created by anaconda on Sat May 12 18:50:26 2018  
	Accessible filesystems, by reference, are maintained under '/dev/disk'  
	See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info  
	/dev/mapper/OtusRoot-LogVol00 /                       xfs     defaults        0 0  
	UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaults        0 0  
	/dev/mapper/OtusRoot-LogVol01 swap                    swap    defaults        0 0  

6. Изменим значение VolGroup00 на OtusRoot в файле /etc/default/grub  
		[root@lvm vagrant]# vi /etc/default/grub  
		[root@lvm vagrant]# cat /etc/default/grub  
	GRUB_TIMEOUT=1  
	GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"  
	GRUB_DEFAULT=saved  
	GRUB_DISABLE_SUBMENU=true  
	GRUB_TERMINAL_OUTPUT="console"  
	GRUB_CMDLINE_LINUX="no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto rd.lvm.lv=OtusRoot/LogVol00 rd.lvm.lv=OtusRoot/LogVol01 rhgb quiet"  
	GRUB_DISABLE_RECOVERY="true"  

7. Изменим значение VolGroup00 на OtusRoot в файле /boot/grub2/grub.cfg  
		[root@lvm vagrant]# vi /boot/grub2/grub.cfg  
		[root@lvm vagrant]# cat /boot/grub2/grub.cfg  
	...  
	`### BEGIN /etc/grub.d/10_linux ###`  
	menuentry 'CentOS Linux (3.10.0-862.2.3.el7.x86_64) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-3.10.0-862.2.3.el7.x86_64-advanced-b60e9498-0baa-4d9f-90aa-069048217fee' {  
	load_video  
	set gfxpayload=keep  
	insmod gzio  
	insmod part_msdos  
	insmod xfs  
	set root='hd0,msdos2'  
	if [ x$feature_platform_search_hint = xy ]; then  
	  search --no-floppy --fs-uuid --set=root --hint='hd0,msdos2'  570897ca-e759-4c81-90cf-389da6eee4cc  
	else  
	  search --no-floppy --fs-uuid --set=root 570897ca-e759-4c81-90cf-389da6eee4cc  
	fi  
	linux16 /vmlinuz-3.10.0-862.2.3.el7.x86_64 root=/dev/mapper/OtusRoot-LogVol00 ro no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto rd.lvm.lv=OtusRoot/LogVol00 rd.lvm.lv=OtusRoot/LogVol01 rhgb quiet  
	initrd16 /initramfs-3.10.0-862.2.3.el7.x86_64.img  
	}  
	if [ "x$default" = 'CentOS Linux (3.10.0-862.2.3.el7.x86_64) 7 (Core)' ]; then default='Advanced options for CentOS Linux>CentOS Linux (3.10.0-862.2.3.el7.x86_64) 7 (Core)'; fi;  
	`### END /etc/grub.d/10_linux ###`  
	...

8. Пересоздаем initrd image, чтобы он знал новое название Volume Group  
		[root@lvm vagrant]# mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)  
	Executing: /sbin/dracut -f -v /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img 3.10.0-862.2.3.el7.x86_64  
	...  
	*** Creating image file ***  
	*** Creating image file done ***  
	*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***  

9. Перезагружаемся и проверяем новое значение Volume Group  
		[root@lvm vagrant]# vgs  
	VG       #PV #LV #SN Attr   VSize   VFree  
	OtusRoot   1   2   0 wz--n- <38.97g    0  

# Решение по добавлению модуля в initrd

1. Создаем папку 01test в каталоге хранения модулей (/usr/lib/dracut/modules.d/).  
		[root@lvm vagrant]# mkdir /usr/lib/dracut/modules.d/01test  

2. Перейдем в созданную папку 01test и создадим в ней скрипт module-setup.sh, который будет  устанавливать модуль и вызывать скрипт test.sh (скопируем в файл текст скрипта любезно предоставленного в методичке)  
		[root@lvm modules.d]# cd 01test  
		[root@lvm 01test]# > module-setup.sh   
		[root@lvm 01test]# vi module-setup.sh   
		[root@lvm 01test]# cat module-setup.sh   
	 #!/bin/bash  

	check() {  
	    return 0  
	}  

	depends() {  
	    return 0  
	}  

	install() {  
	    inst_hook cleanu  p 00 "${moddir}/test.sh"
	}

3. Создадим в каталоге второй скрипт test.sh (также скопируем в файл текст скрипта из методички)  
		[root@lvm 01test]# > test.sh    
		[root@lvm 01test]# vi test.sh    
		[root@lvm 01test]# cat test.sh    
	 #!/bin/bash  

	exec 0<>/dev/console 1<>/dev/console 2<>/dev/console  
	cat <<'msgend'  
	Hello! You are in dracut module!  
	 ___________________  
	< I'm dracut module >  
	 -------------------  
	   \  
	    \  
		.--.  
	       |o_o |  
	       |\_/ |  
	      //   \ \  
	     (|     | )  
	    /'\_   _/`\  
	    \___)=(___/  
	msgend  
	sleep 10  
	echo " continuing...."  


4. Запускаем сборку образа initrd  
		[root@lvm 01test]# mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)  
	Executing: /sbin/dracut -f -v /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img 3.10.0-862.2.3.el7.x86_64  
	...  
	*** Creating image file ***  
	*** Creating image file done ***  
	*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***  

5. Проверим, загружен ли в образ модуль test   
		[root@lvm 01test]# lsinitrd -m /boot/initramfs-$(uname -r).img | grep test  
	test  

6. Выполняем перезагрузку, вручную отключаем опции загрузки rghb и quiet, нажимаем ctrl-x и следим за загрузкой. На определенном этапе видим выполнение скрипта test.sh  

7. Пробуем отключить опции загрузки, исправив для этого файл /boot/grub2/grub.cfg  
		[vagrant@lvm ~]$ sudo su  
		[root@lvm vagrant]# vi /boot/grub2/grub.cfg  

8. Снова выполняем перезагрузку и следим за загрузкой. На определенном этапе опять видим выполнение скрипта test.sh  


