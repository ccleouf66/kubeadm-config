# Cross region vRack network performance tests

## vRack environment
vRack *pcc145,146,3ds* in CA environment.  
The dedicated servers are integrated in the vRack.  
The private network used is *cyril-net*: *VLAN 1907* - *10.119.0.0/16*.  

## Dedicated servers
3 dedicated servers are provisioned, in 3 different regions.  
Each server is *SCALE-3*:

| Component    | Configuration |
| ----------- | ----------- |
| CPU | AMD Epyc 7642 - 48c/96t - 2.3GHz/3.3GHz |
| RAM | 256GB DDR4 ECC 2933MHz |
| System Disk | 2x SSD SATA 480GB Soft RAID |
| Data Disk | 2×1.92 TB SSD NVMe |
| Public Network| 1Gbit/s unmetered and guaranteed |
| Private Network | 25Gbit/s unmetered and guaranteed |

And the 3 servers characteristics:

| Hostname    | Region | Public IP | Private IP |
| ----------- | ----------- | -- | -- |
| lejav-scale3-lim | LIM |162.19.168.100 | 10.119.250.101 |
| lejav-scale3-sbg | SBG |10.119.250.103 | 10.119.250.102 |
| lejav-scale3-rbx | RBX | 141.95.168.7 | 10.119.250.103 |

## OS
All are installed with Ubuntu 22.04.  
OS is upgraded first:

```bash
$ sudo apt update
...

$ sudo apt upgrade -y
...

$ uname -r
5.15.0-47-generic

$ cat /etc/os-release
PRETTY_NAME="Ubuntu 22.04 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04 LTS (Jammy Jellyfish)"
VERSION_CODENAME=jammy
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=jammy
```

## Network configuration
Each server has 4 ports: 2 for public interface and 2 for private one. They are managed in a LACP aggregation on OVH side. The server configuration must be adapted to match the LACP configuration.  
Here is for instance the configuration for *lejav-scale3-lim* server:
```yaml
$ cat /etc/netplan/50-cloud-init.yaml
# This file is generated from information provided by the datasource.  Changes
# to it will not persist across an instance reboot.  To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    version: 2
    ethernets:
      private:
        match:
          name: ens2f*
        mtu: 9000
      public:
        match:
          name: ens3f*
    bonds:
      bond0:
        interfaces: [public]
        parameters:
          mode: 802.3ad
          mii-monitor-interval: 100
          lacp-rate: fast
          down-delay: 200
          transmit-hash-policy: layer3+4
        dhcp4: true
        accept-ra: false
        # use mac address of one interface, else dhcp will not work
        macaddress: 10:70:fd:c5:39:68
      bond1:
        interfaces: [private]
        mtu: 9000
        parameters:
          mode: 802.3ad
          mii-monitor-interval: 100
          lacp-rate: fast
          down-delay: 200
          transmit-hash-policy: layer3+4
    vlans:
      cyril-net:
        id: 1904
        link: bond1
        addresses: [10.119.250.101/16]
```

## Tests methodology
The objective is to measure latency and maximum network performances between the dedicated servers.  

For latency *ping* utility is used, with 1 packet per 0.1 second and 100 packets:
```bash
$ ping -i 0.1 -c 100 10.119.250.10x
PING 10.119.250.10x (10.119.250.10x) 56(84) bytes of data.
64 bytes from 10.119.250.10x: icmp_seq=1 ttl=64 time=3.85 ms
64 bytes from 10.119.250.10x: icmp_seq=2 ttl=64 time=3.75 ms
64 bytes from 10.119.250.10x: icmp_seq=3 ttl=64 time=3.81 ms
64 bytes from 10.119.250.10x: icmp_seq=4 ttl=64 time=3.84 ms
64 bytes from 10.119.250.10x: icmp_seq=5 ttl=64 time=3.74 ms
64 bytes from 10.119.250.10x: icmp_seq=6 ttl=64 time=3.84 ms
64 bytes from 10.119.250.10x: icmp_seq=7 ttl=64 time=3.85 ms
64 bytes from 10.119.250.10x: icmp_seq=8 ttl=64 time=3.80 ms
...
64 bytes from 10.119.250.10x: icmp_seq=94 ttl=64 time=3.77 ms
64 bytes from 10.119.250.10x: icmp_seq=95 ttl=64 time=3.83 ms
64 bytes from 10.119.250.10x: icmp_seq=96 ttl=64 time=3.82 ms
64 bytes from 10.119.250.10x: icmp_seq=97 ttl=64 time=3.84 ms
64 bytes from 10.119.250.10x: icmp_seq=98 ttl=64 time=3.75 ms
64 bytes from 10.119.250.10x: icmp_seq=99 ttl=64 time=3.74 ms
64 bytes from 10.119.250.10x: icmp_seq=100 ttl=64 time=3.73 ms

--- 10.119.250.10x ping statistics ---
100 packets transmitted, 100 received, 0% packet loss, time 9997ms
rtt min/avg/max/mdev = 3.730/3.785/3.947/0.050 ms
```
Average value is used. If there are important discrepancies between min and max, it will be pointed out.

For performance test, *iperf3* is used, with default packet size (128 kB) and for a 30 seconds period (*-t 30*).  
Tests are done with 1, 2, 4, 8 and 16 threads (*-P* option).  
Average value is used. If there are important discrepancies during the test it will be pointed out.

## Tests results with standard kernel tuning
### Network latency (ms)
| From -> To | Average latency (ms) |
| -- | --: |
| RBX -> LIM | 8.9 |
| LIM -> RBX | 8.8 |
| RBX -> SBG | 10.6 |
| SBG -> RBX | 10.5 |
| LIM -> SBG | 3.8 |
| SBG -> LIM | 3.8 |

In conclusion, latencies go up to more than 10 ms for RBX - SBG, with best latency between LIM and SBG.

### Network performance (Gb/s)
| From -> To | 1 thr. | 2 thr. | 4 thr. | 8 thr. | 16 thr. |
| -- | --: | --: | --: | --: | --: |
| RBX -> LIM | 2.7 | 5.4 | 10.8 | 21.3 | 25.5 |
| LIM -> RBX | 2.7 | 5.4 | 10.8 | 21.2 | 21.2 |
| RBX -> SBG | 2.3 | 4.6 | 9.2 | 18.2 | 25.0 |
| SBG -> RBX | 2.3 | 4.6 | 9.2 | 18.2 | 23.0 |
| LIM -> SBG | 6.1 | 12.0 | 24.0 | 24.0 | 24.0 |
| SBG -> LIM | 6.2 | 12.3 | 24.0 | 28.0 | 26.5|

In conclusion the maximum performances are reached with 8 threads. Performances are linear with the number of threads until the maximum bandwidth is reached.  
**But performances are not good at all with only one thread.**  
System monitoring done during the tests show that CPU is not the bottleneck. For 1 thread test, less than 7% CPU thread is used for the sender, and 12% for the receiver.  
Note on maximum bandwidth: obviously QoS shapes the traffic to limit it at 25Gb/s. Thus some times the bandwidth can exceed the 25Gb/s limit and after be shaped below.

## Kernel tuning
The kernel is now tuned to optimize the performances.  
The tuning includes also security improvements.  

Here is the sysctl configuration:
```bash
$ sudo cat /etc/sysctl.d/99-ovh.conf
#
# Security and performance tuning - only IPV4, adapt to IPV6 is needed
# For security, inspired from Google default configuration
#


#
# Security
#

# Turn on SYN-flood protections.  Starting with 2.6.26, there is no loss
# of TCP functionality/features under normal conditions.  When flood
# protections kick in under high unanswered-SYN load, the system
# should remain more stable, with a trade off of some loss of TCP
# functionality/features (e.g. TCP Window scaling).
net.ipv4.tcp_syncookies=1

# Ignore source-routed packets
net.ipv4.conf.all.accept_source_route=0
net.ipv4.conf.default.accept_source_route=0

# Ignore ICMP redirects from non-GW hosts
net.ipv4.conf.all.accept_redirects=0
net.ipv4.conf.default.accept_redirects=0
net.ipv4.conf.all.secure_redirects=1
net.ipv4.conf.default.secure_redirects=1

# Don't pass traffic between networks or act as a router
net.ipv4.ip_forward=0
net.ipv4.conf.all.send_redirects=0
net.ipv4.conf.default.send_redirects=0

# Turn on Source Address Verification in all interfaces to
# prevent some spoofing attacks.
net.ipv4.conf.all.rp_filter=1
net.ipv4.conf.default.rp_filter=1

# Ignore ICMP broadcasts to avoid participating in Smurf attacks
net.ipv4.icmp_echo_ignore_broadcasts=1

# Ignore bad ICMP errors
net.ipv4.icmp_ignore_bogus_error_responses=1

# Log spoofed, source-routed, and redirect packets
net.ipv4.conf.all.log_martians=1
net.ipv4.conf.default.log_martians=1

# Addresses of mmap base, heap, stack and VDSO page are randomized
kernel.randomize_va_space=2

# Reboot the machine soon after a kernel panic.
kernel.panic=30

# Kuberntes requirement: pass bridged IPv4 traffic to iptables chains.
# This is a requirement for Container Network Interface (CNI) plug-ins to work.
# See https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#network-plugin-requirements
net.bridge.bridge-nf-call-iptables=1



#
# Performance
#

# Limit of socket listen() backlog
net.core.somaxconn=4096

# Maximal number of remembered connection requests, which have not received an acknowledgment from connecting client.
net.ipv4.tcp_max_syn_backlog=1024

# The maximum receive / send socket buffer size in bytes.
net.core.rmem_max=33554432
net.core.wmem_max=33554432

# Enable timestamps as defined in RFC1323 with random offset
net.ipv4.tcp_timestamps=1

# Window scaling
net.ipv4.tcp_window_scaling=1
net.ipv4.tcp_wmem=8192 1048576 33554432
net.ipv4.tcp_rmem=8192 1048576 33554432

# Maximum ancillary buffer size allowed per socket
net.core.optmem_max=524287

# Maximum number of packets, queued on the INPUT side, when the interface receives packets faster than kernel can process them.
net.core.netdev_max_backlog=250000

# How often TCP sends out keepalive messages when keepalive is enabled
net.ipv4.tcp_keepalive_time=3000
```
It will be applied after the server reboot.


## Tests results with optimized kernel tuning
Only performance tests are rerun.  

### Network performance (Gb/s)
| From -> To | 1 thr. | 2 thr. | 4 thr. | 8 thr. | 16 thr. |
| -- | --: | --: | --: | --: | --: |
| RBX -> LIM | 14.8 | 29.3 | 28.6 | 23.8 | 27.5 |
| LIM -> RBX | 14.8 | 23.5 | 24.7| 25.4 | 24.5 |
| RBX -> SBG | 12.5 | 24.5 | 25.5 | 22.7 | 22.6 |
| SBG -> RBX | 12.6 | 24.5 | 22.2 | 23.0 | 23.3 |
| LIM -> SBG | 24.7 | 24.7 | 25.0 | 22.5 | 23.7 |
| SBG -> LIM | 24.7 | 29.9 | 24.7 | 28.8 | 25.7 |

For LIM-SBG, maximum bandwith is reached with only one thread.  
**We recommend to systematically tune the kernel to improve the network performances.**  
System monitoring done during the tests show that CPU is not the bottleneck. For 1 thread test for LIM-SBG (maximum performance), around 65% CPU thread is used for the sender, and 75% for the receiver.  
Note on maximum bandwidth: obviously QoS shapes the traffic to limit it at 25Gb/s. Thus some times the bandwidth can exceed the 25Gb/s limit and after be shaped below.


## And more...
These kernel tunings can also be applied for VMs.


