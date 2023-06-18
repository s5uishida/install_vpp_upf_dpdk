# Install VPP-UPF with DPDK on Host
This briefly describes the steps and configuration to build and install [oai-cn5g-upf-vpp](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf-vpp) based on [travelping/upg-vpp](https://github.com/travelping/upg-vpp).
**It is intended to be prepared for use with [Open5GS](https://github.com/open5gs/open5gs) and [free5GC](https://github.com/free5gc/free5gc).**

---

<h2 id="toc">Table of Contents</h2>

- [Simple Overview of VPP-UPF and Data Network Gateway](#overview)
- [Build OAI UPF (VPP-UPF) on VM-UP](#build)
  - [Clone OAI UPF (VPP-UPF)](#clone)
  - [Edit oai-cn5g-upf-vpp/build/scripts/build_helper.upf](#edit)
  - [Install VPP-UPF software dependencies](#depend)
  - [Build VPP-UPF](#build_1)
- [Setup VPP-UPF with DPDK on VM-UP](#setup_up)
  - [Load kernel module "uio_pci_generic"](#load_module)
  - [Install DPDK](#install_dpdk)
  - [Check Interfaces](#check_interfaces)
  - [Bind enp0s9/enp0s10/enp0s16 interfaces to DPDK compatible driver (e.g. uio_pci_generic here)](#bind_interfaces)
  - [Verify DPDK binding](#verify_binding)
  - [Create configuration files](#conf)
- [Run VPP-UPF with DPDK on VM-UP](#run)
  - [Verify interfaces at VPP](#verify)
- [Setup Data Network Gateway on VM-DN](#setup_dn)
- [Changelog (summary)](#changelog)

---

<h2 id="overview">Simple Overview of VPP-UPF and Data Network Gateway</h2>

This describes a simple configuration of VPP-UPF and Data Network Gateway, focusing on U-Plane.
**Note that this configuration is implemented with Virtualbox VMs.**

The following minimum configuration was set as a condition.
- One UPF and Data Network Gateway

The built simulation environment is as follows.

<img src="./images/network-overview.png" title="./images/network-overview.png" width=800px></img>

The VPP-UPF used is as follows.
- VPP-UPF - OpenAir CN 5G for UPF v1.5.1 (2023.06.14) - https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf-vpp

Each VMs are as follows.  
| VM | SW & Role | IP address | OS | CPU<br>(Min) | Memory<br>(Min) | HDD<br>(Min) |
| --- | --- | --- | --- | --- | --- | --- |
| VM-UP | OpenAir CN 5G for UPF | 192.168.0.151/24 | Ubuntu 22.04 | 2 | 8GB | 20GB |
| VM-DN | Data Network Gateway  | 192.168.0.152/24 | Ubuntu 22.04 | 1 | 1GB | 10GB |

The network interfaces of each VM are as follows.
**Note. Do not enable(up) any devices that will be under the control of DPDK.
These devices will be enabled and set IP addresses in the `init.conf` file of VPP-UPF.**
| VM | Device | Network Adapter | IP address | Interface | Under DPDK |
| --- | --- | --- | --- | --- | --- |
| VM-UP | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.151/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.13.151/24 | N3 | x |
| | enp0s10 | NAT Network | 192.168.14.151/24 | N4 | x |
| | enp0s16 | NAT Network | 192.168.16.151/24 | N6 | x |
| VM-DN | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.152/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.16.152/24 | N6 | -- |

NAT networks of Virtualbox  are as follows.
| Network Name | Network CIDR |
| --- | --- |
| N3 | 192.168.13.0/24 |
| N4 | 192.168.14.0/24 |
| N6 | 192.168.16.0/24 |

**Note. Virtualbox GUI tool can only register up to 4 Network Adapters in one VM.
Since 5 Network Adapters are registered in VM-UP, one cannot be registered with the GUI tool.
In this case, directly edit the vbox file as follows and register the remaining Network Adapter.**

**For example)**
```diff
--- upf-vpp-dpdk-10.vbox.orig   2023-06-12 20:53:32.344961102 +0900
+++ upf-vpp-dpdk-10.vbox        2023-06-13 21:57:19.777484821 +0900
@@ -68,7 +68,12 @@
           </DisabledModes>
           <NATNetwork name="N4"/>
         </Adapter>
-        <Adapter slot="8" MACAddress="0800272F0298" cable="false"/>
+        <Adapter slot="8" enabled="true" MACAddress="0800272F0298" type="82540EM">
+          <DisabledModes>
+            <InternalNetwork name="intnet"/>
+          </DisabledModes>
+          <NATNetwork name="N6"/>
+        </Adapter>
         <Adapter slot="9" MACAddress="080027A0C784" cable="false"/>
         <Adapter slot="10" MACAddress="080027C6438A" cable="false"/>
         <Adapter slot="11" MACAddress="080027594595" cable="false"/>
```

Set network instance to `internet`.
| Network Instance |
| --- |
| internet |

<h2 id="build">Build OAI UPF (VPP-UPF) on VM-UP</h2>

Please refer to the following for building OAI UPF (VPP-UPF).
- VPP-UPF - OpenAir CN 5G for UPF v1.5.1 (2023.06.14) - https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf-vpp/-/blob/master/docs/INSTALL_ON_HOST.md

<h3 id="clone">Clone OAI UPF (VPP-UPF)</h3>

```
# git clone https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf-vpp.git
# cd oai-cn5g-upf-vpp
# git checkout develop
```

<h3 id="edit">Edit oai-cn5g-upf-vpp/build/scripts/build_helper.upf</h3>

```diff
--- build_helper.upf.orig       2023-06-11 19:29:54.000000000 +0900
+++ build_helper.upf    2023-06-11 19:32:08.000000000 +0900
@@ -153,6 +153,7 @@
   $SUDO cp -rf $OPENAIRCN_DIR/vpp/build-root/install-vpp-native/vpp/bin/vppctl /bin/
   echo_info "Copied binaries to /bin"
   # Copying necessary libraries
+  $SUDO mkdir /usr/lib/x86_64-linux-gnu/vpp_plugins/
 #  $SUDO cp -rf $OPENAIRCN_DIR/vpp/build-root/install-vpp-native/vpp/lib/vpp_plugins /usr/lib/x86_64-linux-gnu/vpp_plugins/
   $SUDO cp -rf $OPENAIRCN_DIR/vpp/build-root/install-vpp-native/vpp/lib/vpp_plugins/upf_plugin.so /usr/lib/x86_64-linux-gnu/vpp_plugins/
   $SUDO cp -rf $OPENAIRCN_DIR/vpp/build-root/install-vpp-native/vpp/lib/vpp_plugins/dpdk_plugin.so /usr/lib/x86_64-linux-gnu/vpp_plugins/
@@ -161,7 +162,7 @@
   $SUDO cp -rf $OPENAIRCN_DIR/vpp/build-root/install-vpp-native/vpp/lib/libvlibmemory.so.21.01.1 /usr/lib/x86_64-linux-gnu/
   $SUDO cp -rf $OPENAIRCN_DIR/vpp/build-root/install-vpp-native/vpp/lib/libvlib.so.21.01.1 /usr/lib/x86_64-linux-gnu/
   $SUDO cp -rf $OPENAIRCN_DIR/vpp/build-root/install-vpp-native/vpp/lib/libsvm.so.21.01.1 /usr/lib/x86_64-linux-gnu/
-  $SUDO cp -rf $OPENAIRCN_DIR/vpp/build-root/install-vpp-native/vpp/lib/libnat.so.21.01.1 /usr/lib/x86_64-linux-gnu/
+#  $SUDO cp -rf $OPENAIRCN_DIR/vpp/build-root/install-vpp-native/vpp/lib/libnat.so.21.01.1 /usr/lib/x86_64-linux-gnu/
   $SUDO ldconfig
   echo_info "Copied libraries to /usr/lib/x86_64-linux-gnu/"
   if [ $(getent group vpp) ]; then
```

<h3 id="depend">Install VPP-UPF software dependencies</h3>

```
# cd oai-cn5g-upf-vpp/build/scripts
# ./build_vpp_upf -I -f
```

<h3 id="build_1">Build VPP-UPF</h3>

```
# cd oai-cn5g-upf-vpp/build/scripts
# ./build_vpp_upf -c -V 
```

<h2 id="setup_up">Setup VPP-UPF with DPDK on VM-UP</h2>

Please refer to the following for setup VPP-UPF with DPDK.
- VPP-UPF - OpenAir CN 5G for UPF v1.5.1 (2023.06.14) - https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf-vpp/-/blob/master/docs/VPP_UPG_WITH_DPDK.md

<h3 id="load_module">Load kernel module "uio_pci_generic"</h3>

```
# modprobe uio_pci_generic
```

<h3 id="install_dpdk">Install DPDK</h3>

```
# apt install dpdk
```

<h3 id="check_interfaces">Check Interfaces</h3>

```
# lshw -c network -businfo
Bus info          Device      Class       Description
=====================================================
pci@0000:00:03.0  enp0s3      network     82540EM Gigabit Ethernet Controller
pci@0000:00:08.0  enp0s8      network     82540EM Gigabit Ethernet Controller
pci@0000:00:09.0  enp0s9      network     82540EM Gigabit Ethernet Controller
pci@0000:00:0a.0  enp0s10     network     82540EM Gigabit Ethernet Controller
pci@0000:00:10.0  enp0s16     network     82540EM Gigabit Ethernet Controller
```

<h3 id="bind_interfaces">Bind enp0s9/enp0s10/enp0s16 interfaces to DPDK compatible driver (e.g. uio_pci_generic here)</h3>

```
# dpdk-devbind.py -b uio_pci_generic  0000:00:09.0  --force
# dpdk-devbind.py -b uio_pci_generic  0000:00:0a.0  --force
# dpdk-devbind.py -b uio_pci_generic  0000:00:10.0  --force
```

<h3 id="verify_binding">Verify DPDK binding</h3>

```
# lshw -c network -businfo
Bus info          Device      Class       Description
=====================================================
pci@0000:00:03.0  enp0s3      network     82540EM Gigabit Ethernet Controller
pci@0000:00:08.0  enp0s8      network     82540EM Gigabit Ethernet Controller
pci@0000:00:09.0              network     82540EM Gigabit Ethernet Controller
pci@0000:00:0a.0              network     82540EM Gigabit Ethernet Controller
pci@0000:00:10.0              network     82540EM Gigabit Ethernet Controller
```
```
# dpdk-devbind.py -s

Network devices using DPDK-compatible driver
============================================
0000:00:09.0 '82540EM Gigabit Ethernet Controller 100e' drv=uio_pci_generic unused=e1000,vfio-pci
0000:00:0a.0 '82540EM Gigabit Ethernet Controller 100e' drv=uio_pci_generic unused=e1000,vfio-pci
0000:00:10.0 '82540EM Gigabit Ethernet Controller 100e' drv=uio_pci_generic unused=e1000,vfio-pci

Network devices using kernel driver
===================================
0000:00:03.0 '82540EM Gigabit Ethernet Controller 100e' if=enp0s3 drv=e1000 unused=vfio-pci,uio_pci_generic *Active*
0000:00:08.0 '82540EM Gigabit Ethernet Controller 100e' if=enp0s8 drv=e1000 unused=vfio-pci,uio_pci_generic *Active*

No 'Baseband' devices detected
==============================

No 'Crypto' devices detected
============================

No 'DMA' devices detected
=========================

No 'Eventdev' devices detected
==============================

No 'Mempool' devices detected
=============================

No 'Compress' devices detected
==============================

No 'Misc (rawdev)' devices detected
===================================

No 'Regex' devices detected
===========================
```

<h3 id="conf">Create configuration files</h3>

Create `/root/openair-upf` directory and put the configuration files there.

- `/root/openair-upf/startup.conf`

```
unix {
  nodaemon
  log /tmp/vpp.log
  full-coredump
  gid vpp
  interactive
  cli-listen /run/vpp/cli.sock
  exec /root/openair-upf/init.conf
}

api-trace {
  on
}

cpu {
  main-core 0
  corelist-workers 1
}

api-segment {
  gid vpp
}

dpdk {
  dev 0000:00:09.0 {name n3}
  dev 0000:00:0a.0 {name n4}
  dev 0000:00:10.0 {name n6}
}

plugins {
    path /usr/lib/x86_64-linux-gnu/vpp_plugins/
    plugin dpdk_plugin.so { enable }
    plugin upf_plugin.so { enable }
}
```

- `/root/openair-upf/init.conf`

Set network instance to `internet`.
```
set interface ip table n6 0
set interface mtu 9000 n6
set interface ip address n6 192.168.16.151/24
set interface state n6 up

set interface ip table n4 0
set interface mtu 9000 n4
set interface ip address n4 192.168.14.151/24
set interface state n4 up

set interface ip table n3 0
set interface mtu 9000 n3
set interface ip address n3 192.168.13.151/24
set interface state n3 up

ip route add 0.0.0.0/0 table 0 via 192.168.16.152 n6

upf pfcp endpoint ip 192.168.14.151 vrf 0

upf node-id fqdn 192.168.14.151

upf nwi name internet vrf 0

upf specification release 16

upf gtpu endpoint ip 192.168.13.151 nwi internet teid 0x000004d2/2
```

<h2 id="run">Run VPP-UPF with DPDK on VM-UP</h2>

```
# /usr/bin/vpp -c /root/openair-upf/startup.conf
0: vlib_sort_init_exit_functions:160: order constraint fcn 'dns_init' not found
0: vnet_feature_arc_init:271: feature node 'acl-plugin-out-ip6-fa' not found (before 'ip6-dvr-reinject', arc 'ip6-output')
0: vnet_feature_arc_init:271: feature node 'nat44-in2out-output' not found (before 'ip4-dvr-reinject', arc 'ip4-output')
0: vnet_feature_arc_init:271: feature node 'acl-plugin-out-ip4-fa' not found (before 'ip4-dvr-reinject', arc 'ip4-output')
dpdk             [warn  ]: not enough DPDK crypto resources
dpdk/cryptodev   [warn  ]: dpdk_cryptodev_init: Not enough cryptodevs
    _______    _        _   _____  ___ 
 __/ __/ _ \  (_)__    | | / / _ \/ _ \
 _/ _// // / / / _ \   | |/ / ___/ ___/
 /_/ /____(_)_/\___/   |___/_/  /_/    

vpp#  
```

<h3 id="verify">Verify interfaces at VPP</h3>

```
vpp# show hardware-interfaces 
              Name                Idx   Link  Hardware
local0                             0    down  local0
  Link speed: unknown
  local
0: format_dpdk_device:590: rte_eth_dev_rss_hash_conf_get returned -95
n3                                 1     up   n3
  Link speed: 1 Gbps
  Ethernet address 08:00:27:bd:c2:88
  Intel 82540EM (e1000)
    carrier up full duplex mtu 9000 
    flags: admin-up pmd maybe-multiseg tx-offload intel-phdr-cksum rx-ip4-cksum
    Devargs: 
    rx: queues 1 (max 1), desc 1024 (min 32 max 4096 align 8)
    tx: queues 1 (max 1), desc 1024 (min 32 max 4096 align 8)
    pci: device 8086:100e subsystem 8086:001e address 0000:00:09.00 numa 0
    max rx packet len: 16128
    promiscuous: unicast off all-multicast on
    vlan offload: strip off filter off qinq off
    rx offload avail:  vlan-strip ipv4-cksum udp-cksum tcp-cksum vlan-filter 
                       jumbo-frame scatter keep-crc 
    rx offload active: ipv4-cksum jumbo-frame scatter 
    tx offload avail:  vlan-insert ipv4-cksum udp-cksum tcp-cksum multi-segs 
    tx offload active: udp-cksum tcp-cksum multi-segs 
    rss avail:         none
    rss active:        none
    tx burst function: eth_em_xmit_pkts
    rx burst function: eth_em_recv_scattered_pkts
0: format_dpdk_device:590: rte_eth_dev_rss_hash_conf_get returned -95

n4                                 2     up   n4
  Link speed: 1 Gbps
  Ethernet address 08:00:27:37:37:0c
  Intel 82540EM (e1000)
    carrier up full duplex mtu 9000 
    flags: admin-up pmd maybe-multiseg tx-offload intel-phdr-cksum rx-ip4-cksum
    Devargs: 
    rx: queues 1 (max 1), desc 1024 (min 32 max 4096 align 8)
    tx: queues 1 (max 1), desc 1024 (min 32 max 4096 align 8)
    pci: device 8086:100e subsystem 8086:001e address 0000:00:0a.00 numa 0
    max rx packet len: 16128
    promiscuous: unicast off all-multicast on
    vlan offload: strip off filter off qinq off
    rx offload avail:  vlan-strip ipv4-cksum udp-cksum tcp-cksum vlan-filter 
                       jumbo-frame scatter keep-crc 
    rx offload active: ipv4-cksum jumbo-frame scatter 
    tx offload avail:  vlan-insert ipv4-cksum udp-cksum tcp-cksum multi-segs 
    tx offload active: udp-cksum tcp-cksum multi-segs 
    rss avail:         none
    rss active:        none
    tx burst function: eth_em_xmit_pkts
    rx burst function: eth_em_recv_scattered_pkts

n6                                 3     up   n6
  Link speed: 1 Gbps
  Ethernet address 08:00:27:2f:02:98
  Intel 82540EM (e1000)
    carrier up full duplex mtu 9000 
    flags: admin-up pmd maybe-multiseg tx-offload intel-phdr-cksum rx-ip4-cksum
    Devargs: 
    rx: queues 1 (max 1), desc 1024 (min 32 max 4096 align 8)
    tx: queues 1 (max 1), desc 1024 (min 32 max 4096 align 8)
    pci: device 8086:100e subsystem 8086:001e address 0000:00:10.00 numa 0
    max rx packet len: 16128
    promiscuous: unicast off all-multicast on
    vlan offload: strip off filter off qinq off
    rx offload avail:  vlan-strip ipv4-cksum udp-cksum tcp-cksum vlan-filter 
                       jumbo-frame scatter keep-crc 
    rx offload active: ipv4-cksum jumbo-frame scatter 
    tx offload avail:  vlan-insert ipv4-cksum udp-cksum tcp-cksum multi-segs 
    tx offload active: udp-cksum tcp-cksum multi-segs 
    rss avail:         none
    rss active:        none
    tx burst function: eth_em_xmit_pkts
    rx burst function: eth_em_recv_scattered_pkts

upf-nwi-internet                   4     up   upf-nwi-internet
  Link speed: unknown
  GTPU
vpp# 
```
```
vpp# show udp punt 
IPV4 UDP ports punt : 2152, 8805
IPV6 UDP ports punt : 2152
```

<h2 id="setup_dn">Setup Data Network Gateway on VM-DN</h2>

First, uncomment the next line in the `/etc/sysctl.conf` file and reflect it in the OS.
```
net.ipv4.ip_forward=1
```
```
# sysctl -p
```
Next, configure NAPT and routing to N6 IP address of VPP-UPF.
```
# iptables -t nat -A POSTROUTING -s <DN> -j MASQUERADE
# ip route add <DN> via 192.168.16.151 dev enp0s9
```
**Note. Set `<DN>` according to the core network.  
ex) `10.45.0.0/16`**

---
With the above steps, the VPP-UPF environment with DPDK has been constructed.
You will be able to work VPP-UPF with Open5GS and free5GC.
I would like to thank the excellent developers and all the contributors of OpenAir CN 5G for UPF, UPG-VPP and DPDK.

<h2 id="changelog">Changelog (summary)</h2>

- [2023.06.18] Added `upf specification release 16` line in `init.conf`. Along with this, the corresponding description was deleted because the correspondence in the case of Open5GS became unnecessary.
- [2023.06.15] Initial release.
