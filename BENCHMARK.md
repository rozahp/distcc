# Gentoo kernel compilation on low end computers with distcc, pump, ccache and lxd/lxc containers.

## 1. Description

This is a simple benchmarking study of different compilation configurations with distcc, pump, ccache and lxd/lxc containers.

## 2. Background

I am an experienced Linux user, but a beginner with the Gentoo distro. While running emerge and reading the forums, it became apperent that compiling huge packages, like gcc or a DE, is a very time consuming endeavour. This other day gcc emerged for upgrade. Tried to compile it but the computer shutdown because of too high temprature - Yah! (Later I found out about the 'ondemand' processor configuration and added it as default mode.) So I surveyed my options and compiled my results in this little write-up. 

There are no detailed instructions how to install the tools of this setup.

Please feel free to comment, correct or give any suggestions/tweaks.

## 3. Hardware line-up

	Computers:	1 master + 2 slaves

	Master OS:	Gentoo
	Processor:	Intel Core2 Duo 1.8GHz
	Memory:		2G
	Swap:		4G

	Slave OS:	Ubuntu 16.04 + LXD/LXC
	Processor:	Intel Core2 Duo 2.0GHz & 2.13GHz
	Memory:		4G
	Swap:		8G

All computers are connected with wire to the same network switch.

## 4. Software installed

	Gentoo Master:	distcc, ccache

	Ubuntu Slaves: 	LXD (apt-get install lxd zfs) 
			latest gentoo (amd64) containers (lxc images list images:) 

	Containers:	distcc ccache

## 5. Network config

	Ubuntu Slaves:	lxd init: container reachable from the internet=yes

	Gentoo Master:	ip route add [container subnet 1]/24 via [ubuntu slave 1 ip]
			ip route add [container subnet 2]/24 via [ubuntu slave 2 ip]

## 6. Distcc config

### 6.1 Main configuration

Gentoo Master and Slave containers:

	DISTCCD_OPTS=""
	DISTCCD_EXEC="/usr/bin/distccd"
	DISTCCD_PIDFILE="/var/run/distccd/distccd.pid"
	DISTCCD_OPTS="${DISTCCD_OPTS} --port 3632"
	DISTCCD_OPTS="${DISTCCD_OPTS} --log-level notice --log-file /var/log/distccd.log"
	DISTCCD_OPTS="${DISTCCD_OPTS} --allow 192.168.0.0/24 --allow 127.0.0.1"
	DISTCCD_OPTS="${DISTCCD_OPTS} -N 15" 

-- Fix logfile

	touch /var/log/distccd.log
	chown distcc:adm /var/log/distccd.log

-- Start distccd

	rc-update add distccd default
	rc-service distccd start

Comment: If you want localhost to run distcc you have to set "--allow 127.0.0.1" and add 127.0.0.1 in your DISTCC_HOSTS evironment variable. Just setting "localhost" in DISTCC_HOSTS will not trigger distcc on localhost.

For debugging distcc set: export DISTCC_VERBOSE=1

### 6.2 Running the test

You can use the command distcc-config but the best option is setting the DISTCC_HOSTS environment variable.

Gentoo Master:

	export DISTCC_HOSTS="--randomize localhost/2 X.X.X.A Y.Y.Y.A ..."
	export DISTCC_HOSTS="--randomize localhost/2 X.X.X.A,lzo,cpp Y.Y.Y.A,lzo,cpp ..." (for running pump)

Slave containers: -

## 7. Comments about the test

The 'time' command is prefixed to all compile commands. I have not tried to tweak any special variables, like ccache cache size, except the N-value in 'make -jN'. For low end computers, distcc + lxc containers seems to be the best option. I have tried compiling the Gentoo kernel with distcc running on Ubuntu 16.04, but got compiling errors - even with the same gcc version. Installing and configuring a "Gentoo" toolchain on Ubuntu was daungting and would probably mess up my Ubuntu installation. That's why I took the fast and easy way and installed LXD on Ubuntu, downloaded a Gentoo container image. Voila - the toolchain was now identical, and the kernel could compile without errors.

The gentoo containers are running on the 2 Ubuntu "slaves". The 'lxc' value in the table headers below is the amount of containers utilized for each compile round.

The kernel configuration is customized and slimmed for the computer, if you think the compile time is oddly low.

Testing with distcc,pump and ccache together gives alot of warnings like: "... cannot use distcc_pump on already preprocessed file (such as emitted by ccache)". 

As you will see in my results, in this small setup of computers, pump or ccache is not helping getting the numbers down. 

## 8. Things to remember

After each compilation with ccache you have to run 'ccache -c' and 'ccache -C' to empty the cache before next run. And of course run: make clean.

## 9. Emerge

If you are going to use distcc with emerge you have to configure your make.conf:

Example configuration:

	FEATURES="distcc"
	MAKEOPTS="-jN -lM"	(N=your calculated number, M=cores on master computer)
	CFLAGS="... -march=YOUR_PROCESSOR_TYPE", example: -march=core2, NOT: -march=native (!)

## 10. Calculating N (in make -jN)

Default formula for N is: computers * cores * 2 + 1

But when using LXC containers, that formula will not suit that well. So I had to find value that gives the lowest compile time. It's about hammering the distcc-slaves and keeping them busy all the time, but not to much or you will start to DDos yourself. For this setup, N=40 seems to be the peak value for master.

If you get to many "IO Errors" or "failed to distribute" during make, you probably have to lower the N. 

## 11. Future studies

- Could I use my friends computers over the Internet to lower the compile time even further?

- distcc seems to be creating new network connections for every job. If persistent connections could be set up, could that speed things up more?

## 12. The benchmark results

### Compiling with no helpers

	cmd: make

		RUN1		RUN2
	===================================
	real    20m4.354s	20m22.167s
	user    19m31.480s	19m52.460s
	sys     1m2.170s	1m5.460s

	
	cmd: make -j3

		RUN1		RUN2
	==================================
	real    12m49.139s	12m47.887s
	user    22m40.290s	22m41.180s
	sys     1m9.780s	1m11.990s

### Compiling with distcc

	cmd: make -jN CC="distcc"

	lxc/N 	2/20		4/30		6/40		8/45
	==================================================================
	real    6m41.775s	6m20.539s	6m7.839s	6m11.272s
	user    10m31.290s	9m52.990s	9m25.970s	9m25.080s
	sys     1m2.760s	0m59.480s	1m0.650s	1m3.410s

	real    6m40.486s	6m25.932s	6m11.976s	6m9.344s
	user    10m27.920s	9m50.820s	9m34.020s	9m27.780s
	sys     1m4.620s	1m0.890s	1m1.500s	1m2.330s

### Compiling with distcc and pump

	cmd: pump make -jN CC="distcc"

	lxc/N 	2/20		4/30		6/40		8/45
	==================================================================
	real	7m2.199s	6m33.341s	6m59.561s	6m56.275s
	user	9m14.550s	8m21.150s	8m11.450s	7m37.140s
	sys	0m54.030s	0m51.860s	0m52.690s	0m52.400s

	real	7m12.727s	6m43.063s	6m57.651s	6m54.132s
	user	9m32.270s	8m25.900s	8m14.170s	7m35.300s
	sys	0m53.320s	0m53.710s	0m52.620s	0m50.240s

### Compiling make + distcc + ccache 

	cmd: make -jN CC="ccache distcc"

	lxc/N 	2/20		4/30		6/40		8/45
	==================================================================
	real	7m32.975s	6m51.302s	6m39.652s	6m35.960s
	user	11m47.400s	10m29.930s	10m4.540s	9m59.630s
	sys	1m10.380s	1m6.310s	1m8.380s	1m6.750s

	real	7m25.447s	6m54.705s	6m41.620s	6m33.465s
	user	11m35.120s	10m39.680s	10m14.620s	9m53.620s
	sys	1m9.480s	1m6.010s	1m6.840s	1m7.400s

### make + distcc + ccache + pump

	cmd: pump make -jN CC="ccache distcc"

	lxc/N 	2/20		4/30		6/40		8/45
	==================================================================
	real	7m32.727s	6m57.459s	6m40.352s	6m33.930s
	user	11m46.700s	10m37.200s	10m7.870s	9m57.190s
	sys	0m59.440s	0m57.680s	0m57.560s	0m56.050s

	real	7m26.226s	6m53.005s	6m41.323s	6m34.207s
	user	11m35.750s	10m31.770s	10m10.770s	9m53.930s
	sys	0m59.850s	0m57.800s	0m57.900s	0m57.210s

## END
