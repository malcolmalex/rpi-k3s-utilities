# Raspberry Pi k3s Setup

This project will be home to some utilities to make the setup of k3s on a cluster of Raspberry Pi's easier. At the moment, this document describes the manual setup process.

## Overall approach to running k3s on a Raspberry Pi Cluster

1. Download raspbian. Can be found at www.raspberrypi.org/downloads/raspbian. Recommend buster lite for it's small size.
1. Write the image to the sd card. I found it easiest to use etcher from balena. On Ubuntu, the sd card is automatically mounted to ```/media/[username]/boot``` and ```/media/[username]/rootfs```.
1. Eject and reinsert or remount the sd card. Use something like
    ```bash
    df -Hl
    ```
    to list the filesystems. You can see it mounted here on /dev/sdb1 (one for the root, one for boot). This will show some results like the following:
        
        Filesystem      Size  Used Avail Use% Mounted on
        udev             34G     0   34G   0% /dev
        tmpfs           6.8G  2.7M  6.8G   1% /run
        /dev/nvme0n1p6  154G   44G  103G  30% /
        ...
        /dev/sdb2       1.9G  1.1G  701M  60% /media/XXX/rootfs
        /dev/sdb1       265M   41M  224M  16% /media/XXX/boot

1. Modify the files on the SD card for configuration.
    ```bash
    cd /media/XXX/boot
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

    Note that the country code should be your 2-letter ISO code.
    
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
    Nmap scan report for raspberrypi.lan (192.168.1.213)
    Host is up (0.0028s latency).    
    ```

1. ssh into pi and change change passwords, hostname, gpu memory split to the minimum 16MB, and create a static IP address.

    In creating each rpi host, you are likely to face issues with your known host file complaining that the remote rpi host identification has changed. Simple way to deal with that is to execute ssh with the following options. You can also disable host checking overall, or something similar if you feel comfortable doing so.

    ```bash
    ssh -o "UserKnownHostsFile=/dev/null" -o "StrictHostKeyChecking=no" pi@raspberrypi.lan
    ```

    Note for above: for each card you will have implicitly added raspberrypi.lan to your known hosts file for a different raspberry pi, so you will get a message like: 
 
    ```
    @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
    @    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
    @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
    IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
    Someone could be eavesdropping on you right now (man-in-the-middle attack)!
    It is also possible that a host key has just been changed.
    ...
    ECDSA host key for 192.168.1.213 has changed and you have requested strict checking.
    Host key verification failed.
    ```

    Use the recommended ssh command above to avoid this, since we know why the remote key has changed.
    
1. Use raspi-config utility to change the password, hostname, set the GPU memory usage, and 

    ```
    sudo raspi-config
    ```

Reboot.

### Installing k3s

Installing k3s on Raspberry Pi is straightforward once the base images are configured correctly. This involves setup of the master node, adding agent nodes, adding a load balancer to make services available outside the cluster, and adding some storage for cluster use.

#### Setup the Master Node

```
curl -sfL https://get.k3s.io | sh -
```

And to verify status:

```
sudo systemctl status k3s
```

You should see ...

```
● k3s.service - Lightweight Kubernetes
   Loaded: loaded (/etc/systemd/system/k3s.service; enabled; vendor preset: enab
   Active: active (running) since Tue 2019-09-17 17:29:48 BST; 1 day 7h ago
     Docs: https://k3s.io
  Process: 529 ExecStartPre=/sbin/modprobe br_netfilter (code=exited, status=0/S
  Process: 530 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCES
 Main PID: 531 (k3s-server)
    Tasks: 103
   Memory: 101.6M
   CGroup: /system.slice/k3s.service
           ├─ 531 /usr/local/bin/k3s server KillMode=process
           ├─ 565 containerd -c /var/lib/rancher/k3s/agent/etc/containerd/config
           ├─1168 containerd-shim -namespace k8s.io -workdir /var/lib/rancher/k3
           ├─1215 containerd-shim -namespace k8s.io -workdir /var/lib/rancher/k3
           ├─1220 containerd-shim -namespace k8s.io -workdir /var/lib/rancher/k3
           ├─1264 /pause
           ├─1270 /pause
           ├─1276 /pause
           ├─1358 containerd-shim -namespace k8s.io -workdir /var/lib/rancher/k3
           ├─1376 /coredns -conf /etc/coredns/Corefile
           ├─1381 containerd-shim -namespace k8s.io -workdir /var/lib/rancher/k3
           ├─1399 /traefik --configfile=/config/traefik.toml
           ├─1405 containerd-shim -namespace k8s.io -workdir /var/lib/rancher/k3
```


To setup agent nodes, you need to get the node token so the agents know how to join the cluster.

```
sudo cat /var/lib/rancher/k3s/server/node-token
```

Copy this and save it for later use.

#### Setup the Agent Nodes

On each node, run

```
curl -sfL https://get.k3s.io | K3S_TOKEN=<token_from_master_node> K3S_URL=https://<master_ip>:6443 sh -
```

And to verify status, run the following on the master node:

```
sudo kubectl get nodes
```

And you should see something like:

```
pi@rpi-k3s-1:~ $ sudo kubectl get nodes
NAME        STATUS   ROLES    AGE   VERSION
rpi-k3s-1   Ready    master   44h   v1.14.6-k3s.1
rpi-k3s-2   Ready    worker   31h   v1.14.6-k3s.1
```

If you don't use sudo here, at least in the way I've set things up, you get ...

```
WARN[2019-09-19T01:19:46.569593016+01:00] Unable to read /etc/rancher/k3s/k3s.yaml, please start server with --write-kubeconfig-mode to modify kube config permissions 
error: Error loading config file "/etc/rancher/k3s/k3s.yaml": open /etc/rancher/k3s/k3s.yaml: permission denied
```

Repeating this process for each RPi, I was able to get the 5-node cluster set up. One node would not show up, but rebooting solved that problem.

#### Setup remote kubectl

You can ssh into the master or nodes to do various things, but that's a hassle. Better to install kubectl on your local machine, and copy the cluster information from the master node into the local kubeconfig file.

To install kubectl, refer to this page: https://kubernetes.io/docs/tasks/tools/install-kubectl/.

I was able to use the following to successfully install kubectl on my linux box, for example:

```
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version
```

kubectl version will give show you some client information, but it will end with an error of sorts, because there is no cluster configuration file available: 

```
The connection to the server <server-name:port> was refused - did you specify the right host or port?
```

To setup kubectl locally to connect to your pi cluster, you will need to grab the contents of the config from the master, and place it in a file named config on your local machine.

The file you need from your master node: ```/etc/rancher/k3s/k3s.yaml```

The place to put it on local machine: ```~/.kube/config``` (filename is config with no extension)

That file will have localhost:6443 specified as the server. Change that entry to reflect the hostname of your master, and you should be able to run kubectl locally and get 

```
mxa@mxa-dev-linux:~/.kube$ kubectl version
Client Version: version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.0", GitCommit:"2bd9643cee5b3b3a5ecbd3af49d09018f0773c77", GitTreeState:"clean", BuildDate:"2019-09-18T14:36:53Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"14", GitVersion:"v1.14.6-k3s.1", GitCommit:"4cd85f14854d942e9016cc15f399785c103242e9", GitTreeState:"clean", BuildDate:"2019-08-19T16:12+00:00Z", GoVersion:"go1.12.9", Compiler:"gc", Platform:"linux/arm"}
```

If you want to add shell autocompletion and all that, check out this: https://kubernetes.io/docs/tasks/tools/install-kubectl/#optional-kubectl-configurations.

### Resources

Below are some of the primary sources used when compiling these instructions

https://blog.boogiesoftware.com/2019/03/building-light-weight-kubernetes.html

https://blog.alexellis.io/test-drive-k3s-on-raspberry-pi/

https://medium.com/@mabrams_46032/kubernetes-on-raspberry-pi-c246c72f362f




