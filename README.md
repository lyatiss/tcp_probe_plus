# tcp_probe_plus Linux Kernel Module

## License
Please review the:
- "LICENSE" file that is part of this distribution and in the same directory as this file
- The header in the "tcp_probe_plus.c" file
- test

## Description
- Based on the "tcp_probe.c" Linux Kernel Module (LKM) by Stephen Hemminger
- More statistics added by Lyatiss, Inc.
	- rttvar: The Round-Trip Time Variation as per RFC 2988
	- rto: The current retransmit timeout in milliseconds
	- lost: The number of packets currently considered to be lost
	- retrans: The number of packet retransmitted since the connection was established
	- inflight: The number of packets sent but not acknowledged yet
	- frto_counter: A counter to track the Forward RTO-Recovery algorithm as described in RFC 4138
	- rqueue: The number of bytes currently waiting to be read in the socket buffer a.ka. recvq
	- wqueue: The number of bytes currently waiting to be sent in the socket buffer a.k.a. sendq
	- socket_idf: The first sequence number seen for the connection to identify it. It can be used to detect that the connection has been reset and avoid reporting incorrect throughput. 

- More sampling options added by Lyatiss, Inc.
	- There are two options
		- Sample an ACK every sampling period (controlled by the Probe time setting as described below), or
		- Sample an ACK only if the congestion window has changed every sampling period
	- Sampling is done by maintaining a connections table (see the Connection hash table section below for details)
- Add connection termination detection (reported as a magic value in the length field)
	- The support for this functionnality is only available for kernel versions >= 2.6.22. Before that, the instrumented method (tcp_done) was defined as an inline method that can't be instrumented through jprobe. 

## Contents
This repository contains:
- `dkms.conf` Config file for dkms
- `Makefile` Makefile 
- `tcp_probe_plus.c` Modified tcp_probe that does the sampling and collects more statistics (NOTE: Works on Linux kernel versions 2.6 and higher)
- `LICENSE` GPLv2 license

## Building the module
1. Copy all the files in this folder to the target directory on your machine, e.g., `/usr/src/tcp_probe_plus` 
2. (Optional) Install DKMS

	On Debian:

		apt-get install dkms
    
3. Install Linux kernel headers

	On Debian, execute the following commands to determine and then install the correct kernel headers

		ubuntu:/usr/src# uname -a
		Linux ubuntu 3.2.0-4-686-pae #1 SMP Ubuntu i686 GNU/Linux
		ubuntu:/usr/src# apt-get install linux-headers-3.2-0-4-686-pae
	
4. Build and install the LKM

	Using DKMS

		ubuntu@host:/usr/src$ sudo dkms add tcp_probe_plus
	
		Creating symlink /var/lib/dkms/tcp_probe_plus/1.1.6/source ->
        	         /usr/src/tcp_probe_plus-1.1.6

		DKMS: add completed.
		ubuntu@host:/usr/src$ sudo dkms build tcp_probe_plus/1.1.6
		ubuntu@host:/usr/src$ sudo dkms install tcp_probe_plus/1.1.6

	Or from the kernel source

		gentoo tcp_probe_plus # make modules modules_install
		make -C /lib/modules/3.7.10-gentoo/build M=/usr/src/tcp_probe_plus modules
		make[1]: Entering directory `/usr/src/linux-3.7.10-gentoo'
		  CC [M]  /usr/src/tcp_probe_plus/tcp_probe_plus.o
		  Building modules, stage 2.
		  MODPOST 1 modules
		  CC      /usr/src/tcp_probe_plus/tcp_probe_plus.mod.o
		  LD [M]  /usr/src/tcp_probe_plus/tcp_probe_plus.ko
		make[1]: Leaving directory `/usr/src/linux-3.7.10-gentoo'
		make -C /lib/modules/3.7.10-gentoo/build M=/usr/src/tcp_probe_plus modules_install
		make[1]: Entering directory `/usr/src/linux-3.7.10-gentoo'
		  INSTALL /usr/src/tcp_probe_plus/tcp_probe_plus.ko
		  DEPMOD  3.7.10-gentoo
		make[1]: Leaving directory `/usr/src/linux-3.7.10-gentoo'
		gentoo tcp_probe_plus # 
	

## Loading the module
 
	ubuntu@host:~$ sudo modprobe tcp_probe_plus
	ubuntu@host:~$ sudo cat /proc/net/tcpprobe
	2.178670575 10.160.229.127:22 10.2.146.10:65221 80 0x3d58a46e 0x3d58a46e 6 2147483647 524280 43 53 255 0 0 0 0 0 0 0x3d58a44e
	...

## Exported Data

The data collected by the LKM is exported through `/proc/net/tcpprobe` and is formatted using the following code:

 	return scnprintf(tbuf, n,
	 "%lu.%09lu %pI4:%u %pI4:%u %d %#llx %#x %u %u %u %u %u %u %u %u %u %u %u %u %#llx\n",
	 (unsigned long) tv.tv_sec,
	 (unsigned long) tv.tv_nsec,
	 &p->saddr, ntohs(p->sport),
	 &p->daddr, ntohs(p->dport),
	 p->length, p->snd_nxt, p->snd_una,
	 p->snd_cwnd, p->ssthresh, p->snd_wnd, p->srtt,
	 p->rttvar, p->rto, p->lost, p->retrans, p->inflight, p->frto_counter,
	 p->rqueue, p->wqueue, p->socket_idf);

| Field | Description |
| ----- | ------------|
| tv.tv_sec | Seconds since tcpprobe loading |
| tv.tv_nsec | Extra milliseconds since tcpprobe loading |
| saddr | Source Address |
| sport | Source Port |
| daddr | Destination Address |
| dport | Destination Port |
| length | Length of the sampled packet (65535 when this is last sample of a connection)|
| snd_nxt | Sequence number of next packet to be sent |
| snd_una | Sequence number of last unacknowledged packet |
| snd_cwnd | Current congestion window size (in number of packets) |
| ssthresh | Slow-start threshold (in number of packets) |
| snd_wnd | Receive window size (in number of packets) |
| srtt | Smoothed rtt (in ms) |
| rttvar | Standard deviation of the rtt (in ms) |
| rto | Number of retransmit timeout events |
| lost | Number of lost packets |
| retrans | Number of retransmitted packets |
| inflight | Number of packets sent but not yet acked |
| frto_counter | Number of spurious RTO events |
| rqueue | Number of bytes in the socket read queue |
| wqueue | Number of bytes in the socket write queue |
| socket_idf | First sequence number seen for the connection |  
 
## Sysctl interface

This LKM offers a sysctl interface to configure it. 

### Configuration

The following configuration parameters are available:

	ubuntu@host:~$ ls -al /proc/sys/net/lyatiss_cw_tcpprobe/
	total 0
	dr-xr-xr-x 1 root root 0 Mar  6 00:18 .
	dr-xr-xr-x 1 root root 0 Mar  5 18:55 ..
	-r--r--r-- 1 root root 0 Mar  6 00:18 bufsize
	-rw-r--r-- 1 root root 0 Mar  6 00:18 debug
	-rw-r--r-- 1 root root 0 Mar  6 00:18 full
	-r--r--r-- 1 root root 0 Mar  6 00:18 hashsize
	-rw-r--r-- 1 root root 0 Mar  6 00:18 maxflows
	-rw-r--r-- 1 root root 0 Mar  6 00:18 port
	-rw-r--r-- 1 root root 0 Mar  6 00:18 probetime
	-rw-r--r-- 1 root root 0 Mar  6 00:18 purgetime

#### Buffer size

This parameter controls the minimum number of sampled packets that will be read from the `/proc/net/tcpprobe` buffer.

NOTE: The read will block until the specified number of packets are available.

- default is 4096 packets
- x: number of sampled packets the buffer can store

Example:

	ubuntu@host:~$ more /proc/sys/net/lyatiss_cw_tcpprobe/bufsize
	4096
	ubuntu@host:~$ sudo sh -c 'echo 1024 > /proc/sys/net/lyatiss_cw_tcpprobe/bufsize'
	

#### Enable/Disable debug information in kernel messages

This parameter controls the debug level.

- 0: no debug
- 1: debug enabled
- 2: trace enabled

Example:

	ubuntu@host:~$ more /proc/sys/net/lyatiss_cw_tcpprobe/debug
	1
	ubuntu@host:~$ sudo sh -c 'echo 0 > /proc/sys/net/lyatiss_cw_tcpprobe/debug'


#### Sample every ACK packet or only on Congestion Window change

This parameter determines how ACK packets are sampled.

- 0: only sample on cwnd changes
- 1: sample on every ack packet received

Example:

	ubuntu@host:~$ more /proc/sys/net/lyatiss_cw_tcpprobe/full
	1
	ubuntu@host:~$ sudo sh -c 'echo 0 > /proc/sys/net/lyatiss_cw_tcpprobe/full'

#### Connection hash table (maxflows/hashsize)

The memory used by the flow table is controlled by two parameters:

- maxflows: Maximum number of flows tracked by this module.
- hashsize: Size of the hashtable.

##### hashsize

This parameter defines the size of the hashtable.

- default size: automatically calculated - similar to the netfilter connection tracker
- x: size of the hashtable - number of slots that the hashtable has

A linked list is used to track flows that hash to the same slot in the hashtable.

Example:

	ubuntu@host:~$ more /proc/sys/net/lyatiss_cw_tcpprobe/hashsize
	0
	ubuntu@host:~$ sudo sh -c 'echo 16384 > /proc/sys/net/lyatiss_cw_tcpprobe/hashsize'

Max flow (see maxflows) has a default value of 2 million flows (2000000).

The minimum hashtable size is 32 slots. If you explicitly set a lower value, it will be reset to 32.

If you leave the hashtable size to be auto-calculated, then it is based on system memory availability, as coded below, and capped at 16,384 slots.

    /* determine hash size (idea from nf_conntrack_core.c) */
    if (!hashsize) {
      hashsize = (((totalram_pages << PAGE_SHIFT) / 16384)
                    / sizeof(struct hlist_head));
      if (totalram_pages > (1024 * 1024 * 1024 / PAGE_SIZE)) {
        hashsize = 16384;
      }
    }
    if (hashsize < 32) {
      hashsize = 32;
    }
    pr_info("Hashtable initialized with %u buckets\n", hashsize);

##### maxflows

This parameter controls the maximum number of flows that this module will track.

- default: 2,000,000 flows
- x: maximum number of flows to track

Example:

	ubuntu@host:~$ more /proc/sys/net/lyatiss_cw_tcpprobe/maxflows
	2000000
	ubuntu@host:~$ sudo sh -c 'echo 1000000 > /proc/sys/net/lyatiss_cw_tcpprobe/maxflows'

#### Port filtering
	
This parameter controls the port-based filtering of the flows to track.

- 0: no filtering
- x: port to match. This is a single port number. If it matches any of the send or receive port, then the flow will be tracked.

Example:

	ubuntu@host:~$ more /proc/sys/net/lyatiss_cw_tcpprobe/port
	0
	ubuntu@host:~$ sudo sh -c 'echo 5001 > /proc/sys/net/lyatiss_cw_tcpprobe/port'


#### Probe time
	
Upon receiving an ACK, the receive time of the ACK is compared with the receive time of the previous ACK for the connection. If the time difference is equal to or more than the probe time, then this ACK is eligible to be written to `/proc/net/tcpprobe`. The probe time is configurable from user space. The default probe time is 500 ms. This value could be passed as a module initialization parameter or changed using this parameter.

- default is 500 ms
- x: sampling interval

Example:

	ubuntu@host:~$ more /proc/sys/net/lyatiss_cw_tcpprobe/probetime
	500
	ubuntu@host:~$ sudo sh -c 'echo 200 > /proc/sys/net/lyatiss_cw_tcpprobe/probetime'


#### Purge time
	
Every `purgetime` the flows that are not active anymore are removed from the flow table. The purge time is configurable from user space. The default purge time is 300 s. This value could be passed as a module initialization parameter or changed using this parameter.

- default is 300 s
- x: purge time interval

Example:

	ubuntu@host:~$ more /proc/sys/net/lyatiss_cw_tcpprobe/purgetime
	500
	ubuntu@host:~$ sudo sh -c 'echo 200 > /proc/sys/net/lyatiss_cw_tcpprobe/purgetime'


### Statistics

This module offers several statistics about its internal behavior.

	ubuntu@host:~$ more /proc/net/stat/lyatiss_cw_tcpprobe
	Flows: active 4 mem 0K
	Hash: size 4721 mem 36K
	cpu# hash_stat: <search_flows found new reset>, ack_drop: <purge_in_progress ring_full>, 
	conn_drop: <maxflow_reached memory_alloc_failed>, err: <multiple_reader copy_failed>
	Total: hash_stat:      0  25877    151    147, ack_drop:      0      0, 
	conn_drop:      0      0, err:      0      0

Description:

- Flows
	- active: Number of active flows being monitored by the module at present.
	- mem: Total memory used by the flow table to monitor the current set of flows.
- Hash
	- size: Number of slots in the hash table (hashtable size).
	- mem: Total memory used by the hash table.
- hash_stat
	- search_flows: Number of flows looked up so far in the hash table.
	- found: Number of flows found in the hash table.
	- new: Number of new flow entries created so far.
	- reset: Number of flows entries that have been invalidated because the flows have been closed/reset. 
- ack_drop
	- purge_in_progress: Number of ACK packets skipped by this module because flow purging was in progress (NOTE: this requires locking the flow table).
	- ring_full: Number of ACK packets dropped because of a slow reader (NOTE: User space process reading `/proc/net/tcpprobe`)
- conn_drop
	- maxlfow_reached: New flow was skipped because maximum number of flows (2 million by default) has already been reached.
	- memory_alloc_failed: New flow was skipped because module was unable to allocate memory for the new flow entry.
- err
	- multiple_reader: Module detected multiple readers while writing to `/proc/net/tcpprobe`. Note that multiple readers are not supported. Each reader will see only part of the flow.
	- copy_failed: Unable to copy the data to the user-space.
