[![Build Status](https://travis-ci.org/eunyoung14/mtcp.svg?branch=master)](https://travis-ci.org/eunyoung14/mtcp)
[![Build Status](https://scan.coverity.com/projects/11896/badge.svg)](https://scan.coverity.com/projects/eunyoung14-mtcp)

# README #

mTCP is a highly scalable user-level TCP stack for multicore systems. 
mTCP source code is distributed under the Modified BSD License. For 
more detail, please refer to the LICENSE. The license term of io_engine 
driver and ported applications may differ from the mTCP’s.

### PREREQUISITE ###

We require the following libraries to run mTCP.
 - ``libps`` (PacketShader I/O engine library) OR ``libdpdk`` (Intel's DPDK package*) or ``netmap`` driver 
 - ``libnuma``
 - ``libpthread``
 - ``librt``

Compling PSIO/DPDK/NETMAP driver requires kernel headers.
 - For Debian/Ubuntu, try ``apt-get install linux-headers-$(uname -r)``

We have modified the dpdk-17.08 package to export net_device stat data 
(for Intel-based Ethernet adapters only) to the OS. To achieve this, the
dpdk-17.08/lib/librte_eal/linuxapp/igb_uio/ directory was updated. We also modified 
``mk/rte.app.mk`` and ``rte_cpuflags.mk`` files to ease the compilation
process of mTCP applications. We recommend using our package for DPDK
installation. 

### INCLUDED DIRECTORIES ###

mtcp: mtcp source code directory
- mtcp/src: source code
- mtcp/src/include: mTCP’s internal header files
- mtcp/lib: library file
- mtcp/include: header files that applications will use

io_engine: event-driven packet I/O engine (io_engine)
- io_engine/driver - driver source code
- io_engine/lib - io_engine library
- io_engine/include - io_engine header files
- io_engine/samples - sample io_engine applications (not mTCP’s)

dpdk-17.08: Intel's Data Plane Development Kit* (modified)
- dpdk-17.08/...

dpdk: Holds soft links to the compiled dpdk-17.08 include/ and lib/ paths
- dpdk/include - the header files
- dpdk/lib - the libraries that need to be linked

apps: mTCP applications
- apps/example - example applications (see README)
- apps/lighttpd-1.4.32 - mTCP-ported lighttpd (see INSTALL)
- apps/apache_benchmark - mTCP-ported apache benchmark (ab) (see README-mtcp)

util: useful source code for applications

config: sample mTCP configuration files (may not be necessary)


### INSTALL GUIDES ###

mTCP can be prepared in two ways.

***PSIO VERSION***

1. make in io_engine/driver:

      ```# make```
   - check ps_ixgbe.ko
   - please note that psio only runs on linux-2.6.x kernels
    (linux-2.6.32 ~ linux-2.6.38)

2. install the driver:
   
   ```# ./install.py <# cores> <# cores>```
   - refer to http://shader.kaist.edu/packetshader/io_engine/
   - you may need to change the ip address in install.py:46

3. Setup mtcp library:
   
   ```bash
   # ./configure --with-psio-lib=<$path_to_ioengine>
   ## e.g. ./configure --with-psio-lib=`echo $PWD`/io_engine
   # make
   ```
  - By default, mTCP assumes that there are 16 CPUs in your system.
    You can set the CPU limit, e.g. on a 8-core system, by using the following command:
    ```bash
    	   # ./configure --with-psio-lib=`echo $PWD`/io_engine CFLAGS="-DMAX_CPUS=8"
    ```
    Please note that your NIC should support RSS queues equal to the MAX_CPUS value
    (since mTCP expects a one-to-one RSS queue to CPU binding).

   - In case `./configure' script prints an error, run the
    following command; and then re-do step-3 (configure again):

    ```# autoreconf -ivf```
    - check libmtcp.a in mtcp/lib
    - check header files in mtcp/include
    - check example binary files in apps/example

4. Check the configurations in apps/example
   - epserver.conf for server-side configuration
   - epwget.conf for client-side configuration
   - you may write your own configuration file for your application

5. Run the applications!


***DPDK VERSION***

1. Set up Intel's DPDK driver. Please use our version of DPDK.
   We have changed the lib/igb_uio/ submodule and a few makefiles
   in DPDK's ``mk/`` subdirectory. The best method to compile DPDK
   package is to use DPDK's tools/setup.sh script. Please compile
   your package based on your own hardware configuration. We tested
   the mTCP stack on Intel Xeon E5-2690 (x86_64) machine with Intel
   82599 Ethernet adapters (10G). We used the following steps in
   the dpdk-setup.sh script for our setup:
   
     - Press [13] to compile the package
     - Press [16] to install the driver
     - Press [20] to setup 1024 2MB hugepages
     - Press [22] to register the Ethernet ports
     - Press [33] to quit the tool

   - check that DPDK package creates a new directory of compiled
  libraries. For x86_64 machines, the new subdirectory should be
  *dpdk-17.08/x86_64-native-linuxapp-gcc*

   - only those devices will work with DPDK drivers that are listed
  on this page: http://dpdk.org/doc/nics. Please make sure that your
  NIC is compatible before moving on to the next step.

   - recent Linux kernels tend to rename Ethernet interfaces based on
  their PCI addresses. If you are using Intel based Ethernet adapters,
  please rename the interface with a ``dpdk`` prefix. You can do so
  using the following command:
  
    ```bash
	# sudo ifconfig <iface> down
	## where X is an integer value.
	## e.g. sudo ip link set ens786f0 name dpdk0
	# sudo ip link set <iface> name dpdkX
    ```

2. Next bring the dpdk-registered interfaces up. Please use the
   ``setup_iface_single_process.sh`` script file present in ``dpdk-17.08/tools/``
   directory for this purpose. Please change lines 49-51 to change the IP	
   address. Under default settings, run the script as:

     ```# ./setup_iface_single_process.sh 4```

   This sets the IP address of your interfaces as 10.0.x.4.

3. Create soft links for ``include/`` and ``lib/`` directories inside
   empty ``dpdk/`` directory:
   ```bash
      # cd dpdk/
      # ln -s <path_to_dpdk_17_08_directory>/x86_64-native-linuxapp-gcc/lib lib
      # ln -s <path_to_dpdk_17_08_directory>/x86_64-native-linuxapp-gcc/include include
   ```
4. Setup mtcp library:
   ```bash
         # ./configure --with-dpdk-lib=$<path_to_mtcp_release_v3>/dpdk
	 ## And not dpdk-17.08!
	 ## e.g. ./configure --with-dpdk-lib=`echo $PWD`/dpdk
   	 # make
    ```

  - By default, mTCP assumes that there are 16 CPUs in your system.
    You can set the CPU limit, e.g. on a 32-core system, by using the following command:
    ```bash
    	   # ./configure --with-dpdk-lib=$<path_to_mtcp_release_v3>/dpdk CFLAGS="-DMAX_CPUS=32"
    ```
    Please note that your NIC should support RSS queues equal to the MAX_CPUS value
    (since mTCP expects a one-to-one RSS queue to CPU binding).
    
   - In case `./configure' script prints an error, run the
    following command; and then re-do step-4 (configure again):
    
       ```# autoreconf -ivf```
   - checksum offloading in the NIC is now ENABLED (by default)!!!
   - this only works for dpdk at the moment
   - use ```./configure --with-dpdk-lib=`echo $PWD`/dpdk --disable-hwcsum``` to disable checksum offloading.
   - check libmtcp.a in mtcp/lib
   - check header files in mtcp/include
   - check example binary files in apps/example

5. Check the configurations in apps/example
   - epserver.conf for server-side configuration
   - epwget.conf for client-side configuration
   - you may write your own configuration file for your application

6. Run the applications!

***NETMAP VERSION***

See README.netmap for details.

***TESTED ENVIRONMENTS***

mTCP runs on Linux-based operating systems (2.6.x for PSIO) with generic 
x86_64 CPUs, but to help evaluation, we provide our tested environments 
as follows.

Intel Xeon E5-2690 octacore CPU @ 2.90 GHz 32 GB of RAM (4 memory channels)
10 GbE NIC with Intel 82599 chipset (specifically Intel X520-DA2)
Debian 6.0.7 (Linux 2.6.32-5-amd64)

Intel Core i7-3770 quadcore CPU @ 3.40 GHz 16 GB of RAM (2 memory channels)
10 GbE NIC with Intel 82599 chipset (specifically Intel X520-DA2)
Ubuntu 10.04 (Linux 2.6.32-47)

Event-driven PacketShader I/O engine (extended io_engine-0.2)

- PSIO is currently only compatible with Linux-2.6.

We tested the DPDK version (polling driver) with Linux-3.13.0 kernel.

***NOTES***

1. mTCP currently runs with fixed memory pools. That means, the size of
   TCP receive and send buffers are fixed at the startup and does not 
   increase dynamically. This could be performance limit to the large 
   long-lived connections. Be sure to configure the buffer size 
   appropriately to your size of workload.

2. The client side of mTCP supports mtcp_init_rss() to create an 
   address pool that can be used to fetch available address space in 
   O(1). To easily congest the server side, this function should be 
   called at the application startup.

3. The supported socket options are limited for right now. Please refer 
   to the mtcp/src/api.c for more detail.

4. The counterpart of mTCP should enable TCP timestamp.

5. mTCP has been tested with the following Ethernet adapters:

   1. Intel-82598	     ixgbe	      	        (Max-queue-limit: 16)
   2. Intel-82599	     ixgbe			(Max-queue-limit: 16)
   3. Intel-I350             igb   			(Max-queue-limit: 08)
   4. Intel-X710	     i40e			(Max-queue-limit: ~)
 
***FREQUENTLY ASKED QUESTIONS***

1. How can I quit the application?
   - Use ^C to gracefully shutdown the application. Two consecutive 
    ^C (separated by 1 sec) will force quit.

2. My application keeps printing "No route to 0.0.0.0"
   - Try to turn off your network-manager for xge*. The network manager 
    can override the IP configuration set by install.py in PSIO driver.

3. Can I statically set the routing or arp table?
   - Yes, mTCP allows static route and arp configuration. Go to the 
    config directory and see sample_route.conf or sample_arp.conf. 
    Copy and adapt it to your condition and link (ln -s) the config 
    directory to the application directory. mTCP will find 
    config/route.conf and config/arp.conf for static configuration.

***CAUTION***

1. Do not remove I/O driver (```ps_ixgbe/igb_uio```) while running mTCP 
   applications. The application will panic!

2. Use the ps_ixgbe/dpdk driver contained in this package, not the one 
   from some other place (e.g., from io_engine github).

                   Contact: mtcp-user at list.ndsl.kaist.edu
                             April 2, 2015. 
                    EunYoung Jeong <notav at ndsl.kaist.edu>
                    M. Asim Jamshed <ajamshed at ndsl.kaist.edu>