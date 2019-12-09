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

Вариант 3 входа без пароля

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
	#
	# /etc/fstab
	# Created by anaconda on Sat May 12 18:50:26 2018
	#
	# Accessible filesystems, by reference, are maintained under '/dev/disk'
	# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
	#
	/dev/mapper/OtusRoot-LogVol00 /                       xfs     defaults        0 0
	UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaults        0 0
	/dev/mapper/OtusRoot-LogVol01 swap                    swap    defaults        0 0
	[root@lvm vagrant]# 

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
	#
	# DO NOT EDIT THIS FILE
	#
	# It is automatically generated by grub2-mkconfig using templates
	# from /etc/grub.d and settings from /etc/default/grub
	#
	...
	### BEGIN /etc/grub.d/10_linux ###
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
	### END /etc/grub.d/10_linux ###
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




	------------------------------------------------------------------------------------------------
	[
	# Created by anaconda on Sat May 12 18:50:26 2018
	#
	# Accessible filesystems, by reference, are maintained under '/dev/disk'
	# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
	#
	/dev/mapper/VolGroup00-LogVol00 /                       xfs     defaults        0 0
	UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaults        0 0

	0 0v/mapper/VolGroup00-LogVol01 swap                    swap    defaults        ~       


	[vagrant@lvm ~]$ ls /home
	vagrant
	[vagrant@lvm ~]$ -t vboxsf
	bash: -t: command not found
	[vagrant@lvm ~]$ more cat typescript
	cat: No such file or directory
	[vagrant@lvm ~]$ ls
	typescript
	[vagrant@lvm ~]$ more typescript
	Script started on Sun 08 Dec 2019 06:32:05 PM UTC
	[vagrant@lvm ~]$ 
	[vagrant@lvm ~]$ 
	[vagrant@lvm ~]$ vgs
	  WARNING: Running as a non-root user. Functionality may be unavailable.
	  /run/lvm/lvmetad.socket: access failed: Permission denied
	  WARNING: Failed to connect to lvmetad. Falling back to device scanning.
	  /dev/mapper/control: open failed: Permission denied
	  Failure to communicate with kernel device-mapper driver.
	  Incompatible libdevmapper 1.02.146-RHEL7 (2018-01-22) and kernel driver (unkno
	wn version).
	[vagrant@lvm ~]$ sudo su
	[root@lvm vagrant]# 
	[root@lvm vagrant]# 
	[root@lvm vagrant]# vgs
	  VG         #PV #LV #SN Attr   VSize   VFree
	  VolGroup00   1   2   0 wz--n- <38.97g    0 
	[root@lvm vagrant]# 
	[root@lvm vagrant]# 
	[root@lvm vagrant]# vgrename VolGroup00 OtusRoot
	  Volume group "VolGroup00" successfully renamed to "OtusRoot"
	[root@lvm vagrant]# 
	[root@lvm vagrant]# 
	[root@lvm vagrant]# vi /etc/fstab
	[root@lvm vagrant]# vi /etc/fstab
	cat /etc/fstab

	#
	# /etc/fstab
	# Created by anaconda on Sat May 12 18:50:26 2018
	#
	# Accessible filesystems, by reference, are maintained under '/dev/disk'
	# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
	#
	/dev/mapper/OtusRoot-LogVol00 /                       xfs     defaults        0 
	0
	UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaul
	ts        0 0


	GRUB_TIMEOUT=1
	GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
	GRUB_DEFAULT=saved
	GRUB_DISABLE_SUBMENU=true
	GRUB_TERMINAL_OUTPUT="console"
	GRUB_CMDLINE_LINUX="no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnam
	es=0 biosdevname=0 elevator=noop crashkernel=auto rd.lvm.lv=OtusRooVlGroup00LogV[?2--More--m.lv=VolGroup00/LogVol01 rhgb quiet"
	.lvm.lv=VolGroup00/LogVol01 rhgb quiet"
	~                                                                           
	~                                                                     
	~                                                               
	~                                                         
	~                                                   
	~                                             
	~                                       
	~                                       
	~                                             
	~                                                   
	~                                                         
	~                                                               
	~                                                                     
	~                                                                             
	-- INSERT --

	#
	# DO NOT EDIT THIS FILE
	#
	# It is automatically generated by grub2-mkconfig using templates
	--More--

















	"/boot/grub2/grub.cfg"
	 127L, 3607C

	### BEGIN /etc/grub.d/00_header ###
	set pager=1

	if [ -s $prefix/grubenv ]; then
	  load_env
	fi
	if [ "${next_entry}" ] ; then
	   set default="${next_entry}"
	   set next_entry=
	   save_env next_entry
	   set boot_once=true
	else
	   set default="${saved_entry}"
	fi

	if [ x"${feature_menuentry_id}" = xy ]; then
	  menuentry_id_option="--id"
	else
	  menuentry_id_option=""
	fi

	export menuentry_id_option

		insmod video_bochs
		insmod video_cirrus
	  fi
	}

	terminal_output console
	if [ x$feature_timeout_style = xy ] ; then
	  set timeout_style=menu
	  set timeout=1
	# Fallback normal timeout code in case the timeout_style feature is
	# unavailable.
	else
	  set timeout=1
	fi
	### END /etc/grub.d/00_header ###

	### BEGIN /etc/grub.d/00_tuned ###
	set tuned_params=""
	set tuned_initrd=""
	### END /etc/grub.d/00_tuned ###

	### BEGIN /etc/grub.d/01_users ###

			set root='hd0,msdos2'
			if [ x$feature_platform_search_hint = xy ]; then
			  search --no-floppy --fs-uuid --set=root --hint='hd0,
	e759-4c81-90cf-389da6eee4cc
			else
			  search --no-floppy --fs-uui
	eee4cct=root 570897ca-e759-4c81-90cf-389da66
			fi
			linux16 /vmlinuz-3.10.0-862.
	ogVol00 ro no_timer_check coner/VolGroup00-LL
	devname=0 elevator=noo0,115200n8 net.ifnames=0 bioss
	=VolGroup00/Loguto rd.lvm.lv=VolGroup00/LogVol00 rd.lvm.lvv
			initrd16 /initramfs-3.10.0-862.2.3.el7.x86_64.img
	}
	if [ "x$default" = 'CentOS 
	efault='Advanced opt3.el7.x86_64) 7 (Core)' ]; then dd
	4) 7 (Core)';OS Linux>CentOS Linux (3.10.0-862.2.3.el7.x86_66
	 fi;

	### BEGIN /etc/grub.d/20_linux_xen ###
	### END /etc/grub.d/20_linux_xen ###

	--More--
	### END /etc/grub.d/10_linux ###-90aa-069048217fee' {
	if [ "x$default" = 'CentOS 
	efault='Advanced opt3.el7.x86_64) 7 (Core)' ]; then dd
	4) 7 (Core)';OS Linux>CentOS Linux (3.10.0-862.2.3.el7.x86_66
	 fi;

	### BEGIN /etc/grub.d/20_linux_xen ###
	### END /etc/grub.d/20_linux_xen ###

	### BEGIN /etc/grub.d/20_ppc_terminfo ###

	### END /etc/grub.d/20_ppc_terminfo ###

	### BEGIN /etc/grub.d/30_os
	-prober ###

	### BEGIN /etc/grub.d/40_cu
	stom ###
	# This file provides an easy way to add custom menu entries.  Simply type the
	# menu entries you want to add after this comment.  Be careful not t
	# the 'exec tail' line above.
	--More--


	o change

	[root@lvm vagrant]# vi /etc/fstab
	cat /etc/fstab

	#
	# /etc/fstab
	# Created by anaconda on Sat May 12 18:50:26 2018
	#
	# Accessible filesystems, by reference, are maintained under '/dev/disk'
	# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
	#
	/dev/mapper/OtusRoot-LogVol00 /                       xfs     defaults        0 
	0
	UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaul
	ts        0 0
	[root@lvm vagrant]# vi  /boot/grup                    swap    defaults        0 
	b2/grub.cfg         cat

	#
	# DO NOT EDIT THIS FILE
	#
	# It is automatically generated by grub2-mkconfig using templates
	# from /etc/grub.d and settings from /etc/default/grub
	#

	### BEGIN /etc/grub.d/00_header ###
	set pager=1

	if [ -s $prefix/grubenv ]; then
	  load_env
	fi
	if [ "${next_entry}" ] ; then
	   set default="${next_entry}"
	   set next_entry=
	   save_env next_entry
	   set boot_once=true
	else
	   set default="${saved_entry}"
	fi

	if [ x"${feature_menuentry_id}" = xy ]; then
	  menuentry_id_option="--id"
	else
	  menuentry_id_option=""
	fi

	export menuentry_id_option

	if [ "${prev_saved_entry}" ]; then
	  set saved_entry="${prev_saved_entry}"
	  save_env saved_entry
	  set prev_saved_entry=
	  save_env prev_saved_entry
	  set boot_once=true
	fi

	function savedefault {
	  if [ -z "${boot_once}" ]; then
		saved_entry="${chosen}"
		save_env saved_entry
	  fi
	}

	function load_video {
	  if [ x$feature_all_video_module = xy ]; then
		insmod all_video
	  else
		insmod efi_gop
		insmod efi_uga
		insmod ieee1275_fb
		insmod vbe
		insmod vga
		insmod video_bochs
		insmod video_cirrus
	  fi
	}

	terminal_output console
	if [ x$feature_timeout_style = xy ] ; then
	  set timeout_style=menu
	  set timeout=1
	# Fallback normal timeout code in case the timeout_style feature is
	# unavailable.
	else
	  set timeout=1
	fi
	### END /etc/grub.d/00_header ###

	### BEGIN /etc/grub.d/00_tuned ###
	set tuned_params=""
	set tuned_initrd=""
	### END /etc/grub.d/00_tuned ###

	### BEGIN /etc/grub.d/01_users ###
	if [ -f ${prefix}/user.cfg ]; then
	  source ${prefix}/user.cfg
	  if [ -n "${GRUB2_PASSWORD}" ]; then
		set superusers="root"
		export superusers
		password_pbkdf2 root ${GRUB2_PASSWORD}
	  fi
	fi
	### END /etc/grub.d/01_users ###

	### BEGIN /etc/grub.d/10_linux ###
	menuentry 'CentOS Linux (3.10.0-862.2.3.el7.x86_64) 7 (Core)' --class centos --c
	lass gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnuli
	nux-3.10.0-862.2.3.el7.x86_64-advanced-b60e9498-0baa-4d9f-90aa-069048217fee' {
		load_video
		set gfxpayload=keep
		insmod gzio
		insmod part_msdos
		insmod xfs
		set root='hd0,msdos2'
		if [ x$feature_platform_search_hint = xy ]; then
		  search --no-floppy --fs-uuid --set=root --hint='hd0,msdos2'  570897ca-
	e759-4c81-90cf-389da6eee4cc
		else
		  search --no-floppy --fs-uuid --set=root 570897ca-e759-4c81-90cf-389da6
	eee4cc
		fi
		linux16 /vmlinuz-3.10.0-862.2.3.el7.x86_64 root=/dev/mapper/VolGroup00-L
	ogVol00 ro no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnames=0 bios
	devname=0 elevator=noop crashkernel=auto rd.lvm.lv=OtusRoot/LogVol00 rd.lvm.lv=O
	tusRoot/LogVol01 rhgb quiet 
		initrd16 /initramfs-3.10.0-862.2.3.el7.x86_64.img
	}
	if [ "x$default" = 'CentOS Linux (3.10.0-862.2.3.el7.x86_64) 7 (Core)' ]; then d
	efault='Advanced options for CentOS Linux>CentOS Linux (3.10.0-862.2.3.el7.x86_6
	4) 7 (Core)'; fi;
	### END /etc/grub.d/10_linux ###

	### BEGIN /etc/grub.d/20_linux_xen ###
	### END /etc/grub.d/20_linux_xen ###

	### BEGIN /etc/grub.d/20_ppc_terminfo ###
	### END /etc/grub.d/20_ppc_terminfo ###

	### BEGIN /etc/grub.d/30_os-prober ###
	### END /etc/grub.d/30_os-prober ###

	### BEGIN /etc/grub.d/40_custom ###
	# This file provides an easy way to add custom menu entries.  Simply type the
	# menu entries you want to add after this comment.  Be careful not to change
	# the 'exec tail' line above.
	### END /etc/grub.d/40_custom ###

	### BEGIN /etc/grub.d/41_custom ###
	if [ -f  ${config_directory}/custom.cfg ]; then
	  source ${config_directory}/custom.cfg
	elif [ -z "${config_directory}" -a -f  $prefix/custom.cfg ]; then
	  source $prefix/custom.cfg;
	fi
	### END /etc/grub.d/41_custom ###
	[root@lvm vagrant]# 
	[root@lvm vagrant]# 
	[root@lvm vagrant]# 
	[root@lvm vagrant]# mkinitrd -f -v /boot/initramfs-$(uname
	 -r).img $(uname -r)
	Executing: /sbin/dracut -f -v /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img 3.10
	.0-862.2.3.el7.x86_64
	dracut module 'busybox' will not be installed, because command 'busybox' could n
	ot be found!
	dracut module 'crypt' will not be installed, because command 'cryptsetup' could 
	not be found!
	dracut module 'dmraid' will not be installed, because command 'dmraid' could not
	 be found!
	dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-
	3g' could not be found!
	dracut module 'multipath' will not be installed, because command 'multipath' cou
	ld not be found!
	dracut module 'busybox' will not be installed, because command 'busybox' could n
	ot be found!
	dracut module 'crypt' will not be installed, because command 'cryptsetup' could 
	not be found!
	dracut module 'dmraid' will not be installed, because command 'dmraid' could not
	 be found!
	dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-
	3g' could not be found!
	dracut module 'multipath' will not be installed, because command 'multipath' cou
	ld not be found!
	*** Including module: bash ***
	*** Including module: nss-softokn ***
	*** Including module: i18n ***
	*** Including module: drm ***
	*** Including module: plymouth ***
	*** Including module: dm ***
	Skipping udev rule: 64-device-mapper.rules
	Skipping udev rule: 60-persistent-storage-dm.rules
	-Skipping udev rule: 55-dm.rules
	*** Including module: kernel-modules ***
	Omitting driver floppy
	*** Including module: lvm ***
	Skipping udev rule: 64-device-mapper.rules
	Skipping udev rule: 56-lvm.rules
	Skipping udev rule: 60-persistent-storage-lvm.rules
	*** Including module: qemu ***
	*** Including module: resume ***
	*** Including module: rootfs-block ***
	*** Including module: terminfo ***
	*** Including module: udev-rules ***
	Skipping udev rule: 40-redhat-cpu-hotplug.rules
	Skipping udev rule: 91-permissions.rules
	*** Including module: biosdevname ***
	*** Including module: systemd ***
	*** Including module: usrmount ***
	*** Including module: base ***
	*** Including module: fs-lib ***
	*** Including module: shutdown ***
	*** Including modules done ***
	*** Installing kernel module dependencies and firmware ***
	*** Installing kernel module dependencies and firmware done ***
	*** Resolving executable dependencies ***
	*** Resolving executable dependencies done***
	*** Hardlinking files ***
	*** Hardlinking files done ***
	*** Stripping files ***
	*** Stripping files done ***
	*** Generating early-microcode cpio image contents ***
	*** No early-microcode cpio image needed ***
	*** Store current command line parameters ***
	*** Creating image file ***
	*** Creating image file done ***
	*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
	' done ***
	[root@lvm vagrant]# exit
	[vagrant@lvm ~]$ 
	[vagrant@lvm ~]$ 
	[vagrant@lvm ~]$ 
	[vagrant@lvm ~]$ 
	[vagrant@lvm ~]$ 
	[vagrant@lvm ~]$ 
	[vagrant@lvm ~]$ 
	[vagrant@lvm ~]$ 
	[vagrant@lvm ~]$ ls

	#
	# /etc/fstab
	# Created by anaconda on Sat May 12 18:50:26 2018
	#
	# Accessible filesystems, by reference, are maintained under '/dev/disk'
	# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
	#
	/dev/mapper/VolGroup00-LogVol00 /                       xfs     defaults        0 0
	UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaults        0 0

	0 0v/mapper/VolGroup00-LogVol01 swap                    swap    defaults        ~                                                                           
	~                                                                     
	~                                                               
	~                                                         
	~                                                   
	--More--


	"/etc/fstab" 11L, 481C


	#
	# /etc/fstab
	# Created by anaconda on Sat May 12 18:50:26 2018
	#
	# Accessible filesystems, by reference, are maintained under '/dev/disk'
	# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
	#
	--More--per/VolGroup00-LogVol00 /                       xfs     defaults        0 0
	UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaults        0 0

	0 0v/mapper/VolGroup00-LogVol01 swap                    swap    defaults        ~                                                                           
	~                                                                     
	~                                                               
	~                                                         
	~                                                   
	~                                             
	~                                       
	~                                       
	~                                             
														
	-- INSERT --





	--------------------------------------------------------------------------------------
