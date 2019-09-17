# Raspberry Pi SDCARD Creator

As I tried to get a bunch of sd cards made for a Raspberry Pi Kubernetes cluster, I found it frustrating to have to create cards manually. I normally use balena for provisioning raspberry pis, but ran into problems trying to get utilities like systemd working, etc.

## Overview of script work

- [ ] Determine if systemd can be used to automate the process of running on first boot, and all subsequent boots
- [x] Get K3s running on one node of rpi cluster

TODO: create a runonce.sh and runalways.sh, use systemd to look for these and run. Or whatever run-parts.sh is.

Some systemd resources:
- https://learn.adafruit.com/running-programs-automatically-on-your-tiny-computer/systemd-writing-and-enabling-a-service
- https://www.dexterindustries.com/howto/run-a-program-on-your-raspberry-pi-at-startup/

Golang shell commands:
- https://nathanleclaire.com/blog/2014/12/29/shelled-out-commands-in-golang/

> Question: How does Balena etcher write .img files to sd card? How does etcher make it easy to select the correct drive?

## Overall approach to running k3s on a Raspberry Pi Cluster

1. Download version of raspbian. Can be found at www.raspberrypi.org/downloads/raspbian. Recommend buster lite for it's small size.
1. Write the image to the sd card. Easiest to use etcher from balena. Default drive location is /media/...us/boot
1. Eject and reinsert or remount the sd card. Use something like
    ```bash
    df -Hl
    ```
    to list the filesystems. You can see it mounted on something like /dev/sdb1 (one for the root, one for boot). This will show some results like the following:
        
        Filesystem      Size  Used Avail Use% Mounted on
        udev             34G     0   34G   0% /dev
        tmpfs           6.8G  2.7M  6.8G   1% /run
        /dev/nvme0n1p6  154G   44G  103G  30% /
        ...
        /dev/sdb2       1.9G  1.1G  701M  60% /media/mxa/rootfs
        /dev/sdb1       265M   41M  224M  16% /media/mxa/boot

1. Modify the files on the SD card for configuration.
    ```bash
    cd /media/mxa/boot
    ```

    
- Disable autoexpand.

  Update cmdline.txt. Remove the following:
    ```
    init=init=/usr/lib/raspi-config/init_resize.sh
    ```
- Configure cgroups (needed by the docker-like k3s).

  Add the following to cmdline.txt:
    ```
    cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
    ```

- Setup wi-fi.

    Create a file in the boot folder named ```wpa_supplicant.conf```, containing:
    
    ```
    ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev 
    update_config=1 
    ap_scan=1 
    fast_reauth=1 
    country=US 

    network={ 
        ssid="<YOUR SSID>" 
        psk="<YOUR PASSWORD>" 
        id_str="0" 
        priority=100 
    } 
    ```

- Setup ssh

    To set up ssh, just touch a file in the boot partition, and when detected at first startup, SSH will be enabled.

    ```bash
    touch ssh
    ```


Now unmount the root partition, insert into Pi, and power up.
1. Determine subnet the pi is on:
    ```bash
    hostname -I
    ```
1. Use nmap to find the raspberrypi IP or hostname
    ```bash
    nmap -sn 192.168.1.0/24
    ```
   or similar. This tells nmap to list out the devices on the whole subnet (0-255). You will get something like the following:
    ```
    Nmap scan report for raspberrypi.lan (192.168.133.213)
    Host is up (0.0028s latency).    
    ```

1. ssh into pi and change change passwords, hostname, gpu memory split to the minimum 16MB, and create a static IP address.

    In creating each rpi host, you are likely to face issues with your known host file complaining that the remote rpi host identification has changed. Simple way to deal with that is to execute ssh with the following options. You can also disable host checking overall, or something similar if you feel comfortable doing so.

    ```bash
    ssh -o "UserKnownHostsFile=/dev/null" -o "StrictHostKeyChecking=no" pi@raspberrypi.lan
    sudo raspi-config
    ```

    Note for above: for each card you will have implicitly added raspberrypi.lan to your known hosts file for a different raspberry pi, so you will get a message like: 
 
    ```
    @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
    @    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
    @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
    IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
    Someone could be eavesdropping on you right now (man-in-the-middle attack)!
    It is also possible that a host key has just been changed.
    The fingerprint for the ECDSA key sent by the remote host is
    SHA256:E0xlnwJ8KQVR4IL/UaFVqTwFWBPke7URwTWTzqfGsQc.
    Please contact your system administrator.
    Add correct host key in /home/mxa/.ssh/known_hosts to get rid of this message.
    Offending ECDSA key in /home/mxa/.ssh/known_hosts:9
      remove with:
      ssh-keygen -f "/home/mxa/.ssh/known_hosts" -R "192.168.133.213"
    ECDSA host key for 192.168.133.213 has changed and you have requested strict checking.
    Host key verification failed.
    ```

    Use the recommended ssh command above to avoid this, since we know why the remote key has changed.
    
1. Use these menus to 


Reboot.

### Installing k3s
#### Bootstrap the k3s server

curl -sfL https://get.k3s.io | sh -

sudo systemctl status k3s

sudo cat /var/lib/rancher/k3s/server/node-token


### Resources

Some systemd resources:
- https://learn.adafruit.com/running-programs-automatically-on-your-tiny-computer/systemd-writing-and-enabling-a-service
- https://www.dexterindustries.com/howto/run-a-program-on-your-raspberry-pi-at-startup/

Golang shell commands:
- https://nathanleclaire.com/blog/2014/12/29/shelled-out-commands-in-golang/

Golang reading/writing to files:
- https://medium.com/learning-the-go-programming-language/streaming-io-in-go-d93507931185
- https://github.com/diskfs/go-diskfs
- https://github.com/cheggaaa/pb
- http://cavaliercoder.com/blog/downloading-large-files-in-go.html

Creating the filesystem?
- https://github.com/diskfs/go-diskfs



