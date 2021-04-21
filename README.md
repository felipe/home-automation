# Home Automation

This is where I am keeping the configuration of my Raspberry Pi (Cluster) running Home Assistant and Home Bridge through Kubernetes.

The overall idea is to manage everything though the individual services themselves, but have anything related to my family's location managed through HomeKit. Abode, Arlo, and any other system will only know the current 'mode'.

The each node is using [Raspberry Pi OS Lite](https://www.raspberrypi.org/software/operating-systems/) and [K3s](https://k3s.io).

## Hardware

 - 4 x Raspberry Pi 4 (8GB)
 - 1 x FreeAgent Go USB Drive (500GB)
 - 1 x [Google Coral Accelerator](https://coral.ai)

## Setup

### microSD Card

 1. Install Raspberry Pi OS Lite on each card. [download](https://www.raspberrypi.org/software/operating-systems/)
 1. In the card root directory run the following to enable SSH.
     ```$ touch ssh```
 1. Install microSD card on each Pi node.
 1. Boot all Pi nodes.
 1. Find and note DHCP provided IP addresses
     - Rpi3: ```$ arp -na | grep -i b8:27:eb```
     - Rpi4: ```$ arp -na | grep -i 'dc:a6:32\|e4:5f:1'```

### OS

 1. Update the `nodes` file with the IP addresses of each
 1. Copy keys for password-less ssh
    - ```$ brew install ssh-copy-id```
    - ```$ ssh-copy-id pi@x.x.x.x```
 1. Update OS packages
     ```
     $ sudo apt-get update && sudo apt-get dist-upgrade -y
     ```
 1. Wifi is disabled by default, but if not disable `wlan0`
     ```$ sudo ifconfig wlan0 down```
 1. Set cmd items add the following line to `/boot/cmdline.txt`
    ```cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory```
 1. Disable swap ```sudo systemctl disable dphys-swapfile.service```
 1. Disable logs - The original configuration corrupted the SD card due to some power failures and too many writes. I have moved the log configuration to memory, which will clear on reboot, but should be fine.

 ```
 vi /etc/fstab
 tmpfs  /tmp tmpfs  defaults,noatime 0  0
 tmpfs  /var/log tmpfs  defaults,noatime,nosuid,mode=0755,size=100m  0  0
 ```

 1. Set static IP on `eth0` by appending the correct code to `/etc/dhcpcd.conf`. Different IPs per node.
```
interface eth0
arping 192.168.1.1
fallback other     

profile 192.168.1.1
static ip_address=192.168.1.x/24
static routers=192.168.1.1
static domain_name_servers=1.1.1.1

profile other
static ip_address=192.168.1.x/24
```
 1. Change the hostname to `kube-*` by modifying:
   - `/etc/hosts`
   - `/etc/hostname`

 1. Reboot `sudo reboot -n`

#### Server Node

 1. Setup NFS Volume
 
   1. Update OS packages
```
$ sudo apt-get install nfs-kernel-server
$ sudo systemctl enable nfs-server
```
   1. Get UUID of volume
```
sudo blkid
```
   1. Create Mount for the volume
```
mkdir /mnt/cluster
```
   1. Use that ID for a mount on `/etc/fstab`
```
PARTUUID=x-x-x-x-x   /mnt/cluster    ext4    auto,nofail,rw,noatime,user     0       0
```
   1. test it
```
sudo mount -a
```
   1. Create shared mount
```
mkdir -p /mnt/cluster/kube-data
mkdir chmod 777 /mnt/cluster/kube-data
```
   1. Modify export by adding this line to `/etc/exports`
```
/mnt/cluster/kube-data *(rw,all_squash,insecure,async,no_subtree_check,anonuid=1000,anongid=1000)
```
   1. Export pages
```
sudo exportfs -rav
sudo rm /lib/systemd/system/nfs-common.service
sudo systemctl enable rpcbind
sudo systemctl enable nfs-kernel-server
sudo systemctl enable nfs-common
sudo systemctl start rpcbind
sudo systemctl start nfs-kernel-server
sudo systemctl start nfs-common
```

 1. Install k3s
    ```$ curl -sfL https://get.k3s.io | sh -```
 1. Get NODE_TOKEN from `/var/lib/rancher/k3s/server/node-token`

#### Worker Nodes

 1. Install k3s
    ```$ curl -sfL https://get.k3s.io | K3S_URL=https://myserver:6443 K3S_TOKEN=NODE_TOKEN sh -```
 1. Copy `/etc/rancher/k3s/k3s.yaml` from the Server to the Clients


## Configuration

### Pod Install order

 1. `kubectl apply -f provisioning/kilo.yaml`
 1. `kubectl apply -f provisioning/adguard.yaml`
 1. `kubectl apply -f provisioning/homebridge.yaml`

### Modes

**Normalizing Mode Names**

_HomeKit_   | _Abode_      | _Arlo_
------------|--------------|----------
Home / Stay | Home         | Home
Away        | Away         | Armed
Night       | n/a - _Home_ | Night
Off         | Standby      | Disarmed

_Items in italics are the current mapping, but it can be changed_

## Pods

### Homebridge

Homebridge links devices that are not HomeKit enabled to Apple's Home App.

#### Service: Abode

Enables accessing the system modes. Currently there is no way to create new modes in Abode, so `Night` mode is the same as `Home` Mode.

#### Service: Arlo

Enables accessing modes other than `Armed` and `Disarmed`. I have created a `Home` mode (which in the Homekit API is also referred to as `stay`), and a `Night` mode.

### K8s Dashboard

Kubernetes provides a dashboard to keep tabs on the system, in order to install it run the following commands.

 1. `kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml`
 1. `kubectl apply -f provisioning/dashboard-adminuser.yaml `
 1. `kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')`
 1. `kube proxy`
 1. visit: `http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login`
 
More at: https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/

### Frigate NRV

[Frigate](https://github.com/blakeblackshear/frigate) records all interactions where there are humans in the video feeds. This is done by getting the Arlo video feed with the [Python Arlo API](https://github.com/jeffreydwalter/arlo/wiki/Streaming-Video), urls are generated with OpenFAAS functions.

### AdGuard

Runs DHCP and DNS filtering. 

### WireGuard

Provides a VPN layer

## Future Plans or Resources

NFS - https://vitux.com/debian_nfs_server/
DynDNS - https://github.com/timothymiller/cloudflare-ddns
OpenFAAS - https://www.shogan.co.uk/kubernetes/raspberry-pi-kubernetes-cluster-with-openfaas-for-serverless-functions-part-4/
WireGuard + AdGuard - https://codingcoffee.dev/blog/wireguard_on_kubernetes_with_adblocking/
Traefik & K3s - https://levelup.gitconnected.com/a-guide-to-k3s-ingress-using-traefik-with-nodeport-6eb29add0b4b
PiHole - https://greg.jeanmart.me/2020/04/13/self-host-pi-hole-on-kubernetes-and-block-ad/
Ports - https://www.bmc.com/blogs/kubernetes-port-targetport-nodeport/
DHCP - https://github.com/npflan/dhcp
DHCP - https://stackoverflow.com/questions/55184317/deploying-a-dhcpd-server

### Cheats

Reboot pod-- kubectl rollout restart deployment -n homebridge homebridge
Bash into pod -- kubectl exec -it homebridge-6b455f4bb8-bs2wd -n homebridge -- /bin/sh
Proxy into Pod -- kubectl -n homebridge port-forward deployment/homebridge 8080:8581
kubectl -n kube-system port-forward deployment/traefik 8080  // Traefik Dashboard

