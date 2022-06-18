# Installing and using distcc

Install distcc on network computers to speed up compile time.

Comments: testing with pump and ccache did not help much with a small setup.

## 1. Install distcc on Gentoo (master and slaves)

### USE-flags

	USE="... -ipv6" (if you dont want/have ipv6 on your system)

### Install distcc

	emerge --ask sys-devel/distcc

### Configuration

Edit /etc/conf.d/distccd

	DISTCCD_OPTS="${DISTCCD_OPTS} --log-level notice --log-file /var/log/distccd.log"
	DISTCCD_OPTS="${DISTCCD_OPTS} --allow 192.168.0.0/24"
	## DISTCCD_OPTS="${DISTCCD_OPTS} --listen 0.0.0.0" ## DO NOT SET = lots of ERROR IO timeout
	DISTCCD_OPTS="${DISTCCD_OPTS} -N 15"

	DISTCCD_OPTS=""
	DISTCCD_EXEC="/usr/bin/distccd"
	DISTCCD_PIDFILE="/var/run/distccd/distccd.pid"
	DISTCCD_OPTS="${DISTCCD_OPTS} --port 3632"
	DISTCCD_OPTS="${DISTCCD_OPTS} --log-level notice --log-file /var/log/distccd.log"
	DISTCCD_OPTS="${DISTCCD_OPTS} --allow 192.168.0.0/24"
	DISTCCD_OPTS="${DISTCCD_OPTS} -N 15"

### Fix logfile

	touch /var/log/distccd.log
	chown distcc:adm /var/log/distccd.log

### Start distccd

	rc-update add distccd default
	rc-service distccd start

## 2. Specify slaves on master

	distcc-config --set-hosts "localhost/2 X.X.X.A Y.Y.Y.A ..."
	distcc-config --set-hosts "localhost/2 X.X.X.A,lzo,cpp Y.Y.Y.A,lzo,cpp ..." (for running pump)

OR

	export DISTCC_HOSTS="--randomize localhost/2 X.X.X.A Y.Y.Y.A ..."

### Config portage (/etc/portage/make.conf)

	# N = twice the number of total (local + remote) CPU cores + 1
	# M = number of local cores

	MAKEOPTS="-jN -lM"
	FEATURES="distcc"

## 3. Install LXD on Ubuntu

	apt-get install lxd zfs
	lxd init (container reachable from the internet=yes)

Ubuntu OS:	

	lxd init: container reachable from the internet=yes

## 4. Config Gentoo master (LXD)

Gentoo master:	

	ip route add [container subnet 1]/24 via [ubuntu slave 1]
        ip route add [container subnet 2]/24 via [ubuntu slave 2]

## 5. Install and config Gentoo LXC slaves 

Run from (1) on containers.

## 6. Testing on Gentoo master

	export DISTCC_VERBOSE=1
	distcc gcc -c main.c -o main.o
	gcc main.o -o main

## 7. Kernal compile

	# N = twice the number of total (local + remote) CPU cores + 1
	# M = number of local cores
	make -j13 -l2 CC="distcc"

## EOF
