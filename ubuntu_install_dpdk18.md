# Install

Latest 30 day trial build of TheRouter is available only by request.
Please contact us by email info at bisonrouter.com

Here are installation steps for Ubuntu 18.04.

## Install the following utilities and libs:

		apt update
		apt -y install g++ libjemalloc-dev libpcap-dev python libpcre2-8-0 autoconf zlib1g-dev flex byacc
		apt -y install cmake libtool libtool-bin subversion rpm libreadline-dev libnuma-dev libnl-genl-3-dev

## Install Proplib

		wget http://therouter.net/downloads/proplib-0.6.3.tar.xz
		tar xvf ./proplib-0.6.3.tar.xz
		cd ./proplib-0.6.3
		./configure
		make
		make install
		cd ..

## Install libcdb

		wget http://therouter.net/downloads/libcdb.tar.gz
		tar xvf ./libcdb.tar.gz
		cd ./libcdb
		wget http://therouter.net/downloads/libcdb_alexk.patch
		cat libcdb_alexk.patch | patch -p1
		export LIBDIR=lib
		export INCDIR=include
		export DESTDIR=/usr/local
		make all
		make install
		cd ..

Notes: install process ends successfully even if it indicates that there has been the following error:

		make -C man install
		make[1]: Entering directory '/home/alex/libcdb/man'
		make[1]: *** No rule to make target 'install'.  Stop.
		make[1]: Leaving directory '/home/alex/libcdb/man'
		Makefile:5: recipe for target 'install' failed
		make: *** [install] Error 2	

## Install qsbr

		wget http://therouter.net/downloads/libqsbr.tar.gz
		tar xzvf libqsbr.tar.gz
		cd ./libqsbr/src
		wget http://therouter.net/downloads/libqsbr_alexk.patch
		cat libqsbr_alexk.patch | patch -p1
		export LIBDIR=lib
		export INCDIR=include/qsbr
		export DESTDIR=/usr/local
		make all
		make install
		cd ../..

## Install bpfjit

		wget http://therouter.net/downloads/bpfjit.tar.gz
		wget http://therouter.net/downloads/sljit-0.92.tgz
		mkdir /usr/lib64
		tar xzvf ./bpfjit.tar.gz
		tar xzvf ./sljit-0.92.tgz
		cd ./bpfjit/sljit/
		cp -rpn ../../sljit-0.92/* ./
		cd ..

 * Edit ./SPECS/libbpfjit.spec and delete or delete the following lines:

		BuildRequires:>make
		BuildRequires:>libtool

 * Run

		make rpm
		rpm --nodeps -ihv RPMS/x86_64/libbpfjit-0.1-1.x86_64.rpm
		cd ..

## Install NPF

		ln -s /usr/lib/x86_64-linux-gnu/libpcap.so.1.8.1 /usr/lib/x86_64-linux-gnu/libpcap.so.1
		git clone -b conn_limits https://github.com/alexk99/npf
		wget http://therouter.net/downloads/city/city.h
		cp city.h /usr/local/include/
		cd npf/src
		cd libnpf/net
		rm ./npf.h
		ln -s ../../kern/npf.h npf.h
		cd ../..
		export DESTDIR=/
		export LIBDIR=/usr/lib64
		export INCDIR=/usr/local/include
		export MANDIR=/usr/local
		make
		make install
		cd ..

 ## Install city hash

		unset LIBDIR
		unset INCDIR
		unset DESTDIR
		git clone https://github.com/google/cityhash
		cd ./cityhash/
		./configure
		make all check CXXFLAGS="-g -O3"
		make install
		cd ..

## Update system library paths

 Add the following lines to the /etc/ld.so.conf.d/router.conf

		/usr/lib64
		/usr/local/lib
		/usr/lib/x86_64-linux-gnu

 Run

		ldconfig

## Install DPDK

### Configure linux kernel

* Make sure the following options are enabled:

		UIO support
		   Device Drivers -> Userspace I/O drivers -> Generic driver for PCI 2.3 and PCI Express cards
		      symbol UIO_PCI_GENERIC
		
		PROC_PAGE_MONITOR
		   File systems -> Pseudo filesystems -> /proc file system support      
		
		HUGETLBFS
		  File systems -> Pseudo filesystems
		
		HPET and HPET_MMAP
		   Device Drivers -> Character devices -> HPET - High Precision Event Timer

* Turn on linux boot time options:
	- Edit GRUB_CMDLINE_LINUX variable in the /etc/default/grub

			GRUB_CMDLINE_LINUX="intel_idle.max_cstate=1 isolcpus=1,2,3,4,5,6,7,9,10,11,12,13,14,15 default_hugepagesz=2M hugepagesz=2M hugepages=3072"

	- Run

			update-grub

	- Note:
		You might want to isolate a different set of cores or reserve different amount of ram for huge pages 
		depending of the hardware configuration of your server.
		The rule here is that you should isolate all cores you're going to use in the router's traffic forwarding process unless
		the perfomance is not a goal.

* Configure hugepages

	- reboot you machine and check that hugepages are available and free

			grep -i huge /proc/meminfo

	- you should get something like this:

			HugePages_Total:    3072
			HugePages_Free:     3072
			HugePages_Rsvd:        0
			HugePages_Surp:        0
			Hugepagesize:       2048 kB

	- Make a mount point for hugepages

			mkdir /mnt/huge

	- Create a mount point entry in the /etc/fstab

			huge         /mnt/huge   hugetlbfs pagesize=2M   0       0

	- Mount hugepages

			mount huge

### Patch and install DPDK

Download DPDK 18.11.3

		cd /usr/src/
		wget https://fast.dpdk.org/rel/dpdk-18.11.3.tar.xz
		tar xvf dpdk-18.11.3.tar.xz
		cd ./dpdk-stable-18.11.3

Download and apply the patches:

		wget http://therouter.net/downloads/dpdk/patches/18.11.3/bonding_mempool.patch
		wget http://therouter.net/downloads/dpdk/patches/18.11.3/bonding_rx_capabilities.patch
		wget http://therouter.net/downloads/dpdk/patches/18.11.3/librte_ip_frag_add_mbuf_counter.patch
		wget http://therouter.net/downloads/dpdk/patches/18.11.3/rte_timer.patch
		wget http://therouter.net/downloads/dpdk/patches/18.11.3/bonding_tx_capabilities.patch
		wget http://therouter.net/downloads/dpdk/patches/18.11.3/bond_lacp.diff
		wget http://therouter.net/downloads/dpdk/patches/18.11.3/scheduler_fix_port_time_rounding.diff
		
		cat ./bonding_mempool.patch | patch -p1
		cat ./bonding_rx_capabilities.patch | patch -p1
		cat ./librte_ip_frag_add_mbuf_counter.patch | patch -p1
		cat ./rte_timer.patch | patch -p1
		cat ./bonding_tx_capabilities.patch | patch -p1
		cat ./bond_lacp.diff | patch -p1
		cat ./scheduler_fix_port_time_rounding.diff | patch -p1

Run the following commands:		

		make install T=x86_64-native-linuxapp-gcc

## TheRouter

### Download TheRouter 

 Please, contact alex@therouter.net

### Configure DPDK ports

#### Define RTE_SDK variable

		export RTE_SDK=/usr/src/dpdk-stable-18.11.3

Add the above export command to /root/.profile

		nano /root/.profile

#### Load drivers and bind your NICs to DPDK

Edit load_dpdk_drivers.sh script from TheRouter archive's sbin directory
and replace 0000:xx:xx.x values with PCI addresses of your NICs, for examaple 0000:02:00.0.
Run 

		$RTE_SDK/usertools/dpdk-devbind.py --status

to find out NIC's PCI addresses.

After correct PCI addresses of your NIC are added to the load_dpdk_drivers.sh file,
run it and then check that NICs are switched to DPDK mode by running again

		$RTE_SDK/usertools/dpdk-devbind.py --status

This time you should find your NICs in the "Network devices using DPDK-compatible driver" section.

### Run TheRouter

#### Prepare configuration files

For configuration examples and options see the page <a href="/conf_options2.md">Command reference</a>

 - create router.conf

		nano /etc/router.conf

#### Run TheRouter

  Copy files from sbin directory from a TheRouter distributive archive to the /usr/local/sbin directory

		cd <THE_ROUTER_ARCHIVE_DIR>
		cp ./sbin/* /usr/local/sbin/

  Define a bash alias

		alias rcli="ip netns exec tr rcli"

  Add the alias to /root/.profile so it's created everytime you log in.

		nano /root/.profile

  Edit /usr/local/sbin/therouter_start.sh script
  and use your NIC's addresses as values for -w parameters. If you have only
  one NIC then delete the second -w parameter.
  
  Start TheRouter

		therouter_start.sh /etc/router.conf

  Check the syslog to ensure that TheRouter has started successfully.

 		ROUTER: router configuration file '/etc/router.conf' successfully loaded  

  Use the 'rcli' alias which will execute an rcli utility from the archive to configure and control TheRouter

		# rcli sh uptime
		Uptime: 0 day(s), 1 hour(s), 38 minute(s), 14 sec(s)

