# Home Automation

This is where I am keeping the configuration of my Raspberry Pi (Cluster) running Home Assistant and Home Bridge through Kubernetes.

The overall idea is to manage everything though the individual services themselves, but have anything related to my family's location managed through HomeKit. Abode, Arlo, and any other system will only know the current 'mode'.

The each node is using [Raspberry Pi OS Lite](https://www.raspberrypi.org/software/operating-systems/) and [K3s](https://k3s.io).

## Hardware

 - 4 x Raspberry Pi 4 (8GB)
 - 1 x FreeAgent Go USB Drive (500GB)
 - 1 x [Google Coral Accelerator](https://coral.ai)

## Setup

## microSD Card

 1. Install Raspberry Pi OS Lite on each card. [download](https://www.raspberrypi.org/software/operating-systems/)
 1. In the card root directory run the following to enable SSH.
     ```$ touch ssh```
 1. Install microSD card on each Pi node.
 1. Boot all Pi nodes.
 1. Find and note DHCP provided IP addresses
     - Rpi3: ```$ arp -na | grep -i b8:27:eb``` 

## OS

 1. Update the `nodes` file with the IP addresses of each
 1. Copy keys for password-less ssh
    - ```$ brew install ssh-copy-id```
    - ```$ ssh-copy-id pi@x.x.x.x```
 1. Update OS packages
     ```$ sudo apt update```
 1. Disable `wlan0`
     ```$ sudo ifconfig wlan0 down```
 1. Set cmd items add the following line to `/boot/cmdline.txt`
    ```cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory```
 1. Set static IP on `eth0` by appending the correct code to `/etc/dhcpcd.conf`. Different IPs per node.
     ```
     interface eth0
     arping 192.168.1.1
     fallback other     
     
     profile 192.168.1.1
     static ip_address=192.168.1.100/24
     static routers=192.168.1.1
     static domain_name_servers=1.1.1.1
     
     profile other
     static ip_address=192.168.1.100/24
     ```
 1. Change the hostname to `kube-*` by modifying:
   - `/etc/hosts`
   - `/etc/hostname`

 1. Reboot `sudo reboot -n`

### Primary Node

 1. Install k3s
     ```$ curl -sfL https://get.k3s.io | sh -```

### Worker Node
 

## Home Assistant

### Service Logs

The original configuration corrupted the SD card due to some power failures and too many writes. I have moved the log configuration to memory, which will clear on reboot, but should be fine. This is done inside the hassio docker image and in the host system (if possible).

```
vi /etc/fstab
tmpfs  /tmp tmpfs  defaults,noatime 0  0
tmpfs  /var/log tmpfs  defaults,noatime,nosuid,mode=0755,size=100m  0  0
```

### Device Metric Logs

Device metrics are sent to DataDog for reporting and alerting. DataDog is set up through a [DockerImage](https://github.com/BenevolentCoders/rpi-datadog-agent).

```
docker run \
  -d --restart unless-stopped \
  --name dd-agent \
  --net=host \
  -h 'hostname' \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -v /proc/:/host/proc/:ro \
  -v /sys/fs/cgroup/:/host/sys/fs/cgroup:ro \
  -e API_KEY=DATADOG_API_KEY \
  benevolentcoders/rpi-datadog-agent
```

## Homebridge

Homebridge is set up through a [Docker Image](https://github.com/oznu/docker-homebridge/wiki/Homebridge-on-Raspberry-Pi).

```
docker run \
  -d --restart unless-stopped \
  --net=host \
  --name=homebridge \
  -e PUID=1000 -e PGID=1000 \
  -e TZ=America/Denver \
  -e HOMEBRIDGE_CONFIG_UI=1 \
  -e HOMEBRIDGE_CONFIG_UI_PORT=8080 \
  oznu/homebridge
```

**Normalizing Mode Names**

_HomeKit_   | _Abode_      | _Arlo_
------------|--------------|----------
Home / Stay | Home         | Home
Away        | Away         | Armed
Night       | n/a - _Home_ | Night
Off         | Standby      | Disarmed

_Items in italics are the current mapping, but it can be changed_

### Abode

Enables accessing the system modes. Currently there is no way to create new modes in Abode, so `Night` mode is the same as `Home` Mode.

### Arlo

Enables accessing modes other than `Armed` and `Disarmed`. I have created a `Home` mode (which in the Homekit API is also referred to as `stay`), and a `Night` mode.

## WIP: Mosquitto

```
docker run \
  --restart=always \
  --name=mqtt \
  --net=host \
  -tid \
  -v ~/mqtt/config:/mqtt/config:ro \
  -v ~/mqtt/log:/mqtt/log \
  -v ~/mqtt/data/:/mqtt/data/ \
  toke/mosquitto
```
