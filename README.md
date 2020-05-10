# Home Automation

This is where I am keeping the configuration of my Raspberry Pi running Home Assistant and Home Bridge.

The overall idea is to manage everything though the individual services themselves, but have anything related to my family's location managed through HomeKit. Abode, Arlo, and any other system will only know the current 'mode'.

The system runs HassOS which has Home Assistant (hass.io) running as a docker image. We will use that same host to run Home Bridge as another docker image next to it. The main install instructions are [here](https://www.home-assistant.io/getting-started/).

### HassOS

To access the host OS you need to add an `authorized_keys` files in the root of a USB drive. Instructions are [here](https://developers.home-assistant.io/docs/en/hassio_debugging.html#hassos-based-hassio).

## Home Assistant

### Service Logs

The original configuration corrupted the SD card due to some power failures and too many writes. I have moved the log configuration to memory, which will clear on reboot, but should be fine.

```
sudo nano /etc/fstab
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
