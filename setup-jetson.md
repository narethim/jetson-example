# Setup Jetson TX2 and Jetson Nano

Set up 3 NVIDIA Jetsons for network testing.

## Pre-requisite

* Docker is installed. ( `docker --version` )
* k3s not installed. ( Remove it if it is installed )
* The Jetson board's ethernet cable is connected to the switch where the RHEL 7 is connected to.

## Setup Jetson TX2

### 1. Make sure that the Docker Daemon file has the right `flannel` subnet ip address for `docker0`

```sh
sudo vim /etc/docker/daemon.json
```

Add the following content:

```sh
{
    "bip":"10.42.0.1/24",
    "mtu": 1450
}
```

Restart `docker`

```sh
sudo service docker restart
```

### 2. Change IP and gateway for `eth0`

```sh
sudo vim /etc/network/interfaces
```

Add the following content:

#### 2.1 jet-b (10.161.29.30)

```sh
    source-directory /etc/network/interfaces.d

    auto lo
        iface lo inet loopback

    auto eth0
        iface eth0 inet static
        address 10.161.29.30
        netmask 255.255.255.0
        gateway 10.161.29.20
```

#### 2.2 jet-nano (10.161.30.30)

```sh
    source-directory /etc/network/interfaces.d

    auto lo
        iface lo inet loopback

    auto eth0
        iface eth0 inet static
        address 10.161.30.30
        netmask 255.255.255.0
        gateway 10.161.30.20
```

#### 2.3 jet-j (10.161.31.30)

```sh
    source-directory /etc/network/interfaces.d

    auto lo
        iface lo inet loopback

    auto eth0
        iface eth0 inet static
        address 10.161.31.30
        netmask 255.255.255.0
        gateway 10.161.31.20
```

### 3. Change resolv.conf

```sh
sudo vim /etc/resolvconf/resolv.conf.d/head
```

Add the following content:

#### 3.1 jet-b (10.161.29.30)

```sh
nameserver 10.161.6.247

nameserver 10.161.29.10
nameserver 10.161.29.20
```

#### 3.2 jet-nano (10.161.30.30)

```sh
nameserver 10.161.6.247

nameserver 10.161.30.10
nameserver 10.161.30.20
```

#### 3.3 jet-j (10.161.31.30)

```sh
nameserver 10.161.6.247

nameserver 10.161.31.10
nameserver 10.161.31.20
```

```sh
sudo vim /etc/resolvconf/resolv.conf.d/base
```

Remove `ga.com` from it:

```sh
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

Reboot it

```sh
sudo reboot
```

### 4. Copy files from RHEL 7.6 to Jetson

```sh
scp -r /home/imnare/Desktop/kubernetes-apps nvidia@10.161.29.30:/home/nvidia/Desktop
scp -r /home/imnare/Desktop/kubernetes-apps nvidia@10.161.30.30:/home/nvidia/Desktop
scp -r /home/imnare/Desktop/kubernetes-apps nvidia@10.161.31.30:/home/nvidia/Desktop

scp -r /home/imnare/Desktop/k3s-0.3.0 nvidia@10.161.29.30:/home/nvidia/Desktop
scp -r /home/imnare/Desktop/k3s-0.3.0 nvidia@10.161.30.30:/home/nvidia/Desktop
scp -r /home/imnare/Desktop/k3s-0.3.0 nvidia@10.161.31.30:/home/nvidia/Desktop
```

### 5. Load docker images on each Jetson node

```sh
cd ~/Desktop/k3s-0.3.0/'docker images'

docker load -i traefik.tar
docker load -i coredns-coredns.tar
docker load -i k8s-gcr-io-pause.tar
docker load -i rancher-klipper-helm.tar
docker load -i rancher-klipper-lb.tar
```

### 6. Copy k3s_offline installer script

```sh
sudo cp /home/nvidia/Desktop/k3s-0.3.0/k3s_offline /usr/local/bin/k3s
sudo chmod +x /usr/local/bin/k3s
```

### 7. Install k3s master and worker node

#### 7.1 Install k3s server on master node

Start k3s master node

```sh
sudo k3s server --docker --resolv-conf /etc/resolv.conf
```

Print out node-token

```sh
sudo cat /var/lib/rancher/k3s/server/node-token
```

#### 7.1 Install k3s agent on worker node

Start k3s worker node

```sh
sudo k3s agent --docker --resolv-conf /etc/resolv.conf --server https://<server_ip_addrsss>:6443 \
  --token <server_node_token>
```

Example:

```sh
sudo k3s agent --docker --resolv-conf /etc/resolv.conf --server https://10.161.30.30:6443 \
  --token <server_node_token>
```

### 8. Label k3s master and worker nodes

On master node `nano-a` issue the following command:

```sh
# Label `nano-a` as master
kubectl label node nano-a kubernetes.io/role=master
kubectl label node nano-a node-role.kubernetes.io/master=""

# Label `jet-b` as node
kubectl label node jet-b kubernetes.io/role=node
kubectl label node jet-b node-role.kubernetes.io/node=""

# Label `jet-b` as node
kubectl label node jet-j kubernetes.io/role=node
kubectl label node jet-j node-role.kubernetes.io/node=""

```
