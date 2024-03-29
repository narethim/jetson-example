# Setup k3s and ns-3 on RHEL 7.6 and Ubuntu 16.04 VM

Setup up RHEL 7.6 with interface alias and Ubuntu 16.04 VM for network testing.

## Pre-requisite

* A physical `RHEL 7.6` system
* Virtualbox installed on `RHEL 7.6` system
* The Jetson board's ethernet cable is connected to the switch where the `RHEL 7.6` system is connected to.

## Create an ubuntu 16.04 VM with 4 bridged network dapters

Record MAC Addresses:

* [ ] Adapter 1 MAC address: 08 00 27 20 e8 37
* [ ] Adapter 2 MAC address: 08 00 27 b3 a5 82
* [ ] Adapter 3 MAC address: 08 00 27 7f d9 0c
* [ ] Adapter 4 MAC address: 08 00 27 dc 60 80

## Clone repository files ( RHEL 7.6 )

Clone repository into Desktop

```sh
cd ~/Desktop

git clone https://imnare@emsbitbucket.ga.com/scm/~phamro/acc-setup.git
```

## Create network interface aliases ( RHEL 7.6 )

Create network interface aliases `em1:1`, `em1:2`, and `em1:3`.

### Create network interface alias `em1:1`

```sh
cd /etc/sysconfig/network-scripts
sudo vim ifcfg-em1:1
```

Add the following content:

```sh
NAME="em1:1"
DEVICE="em1:1"
ONBOOT="yes"
NETBOOT="yes"
IPV6INIT="no"
BOOTPROTO=none
IPADDR="10.161.29.10"
NETMASK="255.255.255.0"
#GATEWAY="10.161.29.20"
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
ZONE=
```

Start the network interface alias up

```sh
sudo ifup em1:1
```

### Create network interface alias `em1:2`

```sh
cd /etc/sysconfig/network-scripts
sudo vim ifcfg-em1:2
```

Add the following content:

```sh
NAME="em1:2"
DEVICE="em1:2"
ONBOOT="yes"
NETBOOT="yes"
IPV6INIT="no"
BOOTPROTO=none
IPADDR="10.161.30.10"
NETMASK="255.255.255.0"
#GATEWAY="10.161.30.20"
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
ZONE=
```

Start the network interface alias up

```sh
sudo ifup em1:2
```

### Create network interface alias `em1:3`

```sh
cd /etc/sysconfig/network-scripts
sudo vim ifcfg-em1:3
```

Add the following content:

```sh
NAME="em1:3"
DEVICE="em1:3"
ONBOOT="yes"
NETBOOT="yes"
IPV6INIT="no"
BOOTPROTO=none
IPADDR="10.161.31.10"
NETMASK="255.255.255.0"
#GATEWAY="10.161.31.20"
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
ZONE=
```

Start the network interface alias up

```sh
sudo ifup em1:3
```

## Create network interfaces and assign static ip addresses ( Ubuntu 16.04 VM )

| Interfaces | Ip address   |
|------------|--------------|
| enp0s3     | 10.161.6.x   |
| enp0s8     | 10.161.29.20 |
| enp0s9     | 10.161.30.20 |
| enp0s10    | 10.161.31.20 |

Modify `/etc/network/interfaces` to make static ip addresses.

```sh
sudo vim /etc/network/interfaces
```

Add the following content:

```sh
source-directory /etc/network/interfaces.d

auto lo
    iface lo inet loopback

auto enp0s8
    iface enp0s8 inet static
    address 10.161.29.20
    netmask 255.255.255.0

auto enp0s9
    iface enp0s9 inet static
    address 10.161.30.20
    netmask 255.255.255.0

auto enp0s10
    iface enp0s10 inet static
    address 10.161.31.20
    netmask 255.255.255.0

```

Restart VM to reflect changes.

```sh
sudo reboot
```

## Add static routes to route traffics to the flannel network of the Jetson boards ( Ubuntu 16.04 VM )

```sh
netstat -rn > route-table-01.txt

sudo ip route add 10.42.0.0/24 via 10.161.29.30 dev enp0s8
sudo ip route add 10.42.0.0/24 via 10.161.30.30 dev enp0s9
sudo ip route add 10.42.0.0/24 via 10.161.31.30 dev enp0s10

netstat -rn > route-table-02.txt

```

Turn on promiscuous mode for network interfaces.

```sh
sudo ifconfig enp0s8 promisc
sudo ifconfig enp0s9 promisc
sudo ifconfig enp0s10 promisc
```

## IP forwarding ( Ubuntu 16.04 VM )

Notes about ip forwarding. When ip forwarding is turned on in this VM, the boards will be able to communicate with each other. This is useful when debugging, but should be turned off when NS-3 simulation is running. The NS-3 routes the boards's traffics.

Turn `on` ip forwarding.

```sh
sudo sysctl -w net.ipv4.ip_forward=1
```

Check status of ip forwarding.

```sh
sudo cat /proc/sys/net/ipv4/ip_forward
```

At this time connectivities between nodes is as follow:

|            | jet-b   | jet-nano | jet-j |
|------------|---------|----------|-------|
| jet-b      |   yes   |    yes   |   yes |
| jet-nano   |   yes   |    yes   |   yes |
| jet-j      |   yes   |    yes   |   yes |

## Build and run NS-3 script ( Ubuntu 16.04 VM )

### Turn off ip_forwarding

Turn `off` ip forwarding.

```sh
sudo sysctl -w net.ipv4.ip_forward=0
```

At this time connectivities between nodes is as follow:

|            | jet-b   | jet-nano | jet-j |
|------------|---------|----------|-------|
| jet-b      |   yes   |    no    |   no  |
| jet-nano   |   no    |    yes   |   no  |
| jet-j      |   no    |    no    |   yes |

### Run the NS-3 application

Run the application: `"scratch/three-node-cluster"`

```sh
cd ~/projects/ns3/tarballs/ns-allinone-3.29/ns-3.19

sudo ./waf --run "scratch/three-node-cluster"
```

Run `ping` test from one Jetson board to another to test connectivity.

From jet-b (10.161.29.30)

```sh
# ping jet-nano
ping -c5 10.161.30.30

# ping jet-j
ping -c5 10.161.31.30
```

From jet-nano (10.161.30.30)

```sh
# ping jet-b
ping -c5 10.161.29.30

# ping jet-j
ping -c5 10.161.31.30
```

From jet-j (10.161.31.30)

```sh
# ping jet-b
ping -c5 10.161.29.30

# ping jet-nano
ping -c5 10.161.30.30
```

|            | jet-b   | jet-nano | jet-j |
|------------|---------|----------|-------|
| jet-b      |    yes  |          |       |
| jet-nano   |         |     yes  |       |
| jet-j      |         |          |   yes |
