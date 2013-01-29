---
layout: post
title: "Решение некоторых проблем в DRBD"
date: 2013-01-29 14:39
comments: true
categories: [drbd, linux, cluster] 
---
### DRBD Diskless

При выходе из строя дискового массива и его восстановления  можно получить следующую ситуацию в drbd:

	# drbd-overview
	2:r2 Connected Secondary/Primary Diskless/UpToDate C r----

#### Для решения этой проблемы необходимо сбросить мета данные:
На активной ноде необходимо отмонтировать раздел. Затем на неактивной ноде отключаем ресурс: 

	# drbdadm down r2

Создаем заново блок мета-данных:

	# drbdadm create-md r2
	 ...
	New drbd meta data block successfully created.

Включаем ресурс обратно (должна начаться синхронизация):

	# drbdadm up r2
	# drbdadm connect r2
	# drbd-overview 
	2:r2 SyncTarget Secondary/Primary Inconsistent/UpToDate C r---- 
	[>....................] sync'ed: 0.1% (31796/31796)M queue_delay: 0.0 ms 

Для изменения скорости синхронизации можно ввести:

	/# drbdsetup /dev/drbd2 syncer -r 10M

Для задания в настройках необходимо прописать в /etc/drbd.conf :

		syncer {
    		rate 100M;
		}

### DRBD Split-brain

Ещё бывает ситуация когда ноды не синхронизируются со следующими признаками:

	# drbd-overview
	3:just StandAlone Primary/Unknown UpToDate/DUnknown r----- /mnt/Just ext3 5.0G 156M 4.6G 4%

Причиной этого может стать состояние split-brain. Для решения этой проблемы необходимо:
На secondary:
	# drbdadm disconnect just
	# drbdadm -- --discard-my-data connect just

На primary:
	# drbdadm connect just

После синхронизации (если она необходима) всё должно работать:

	# drbd-overview
	3:just Connected Primary/Secondary UpToDate/UpToDate C r----- /mnt/Just ext3 5.0G 156M 4.6G 4%
