---
layout: post
title: "K3s on Raspberry Pi - Initial Setup"
description: ""
category: "Development"
tags: [kubernetes,k3s,raspberry-pi]
---
{% include JB/setup %}

As part of trying to learn more about Kubernetes, I thought it'd be interesting to setup a mini cluster running on
Raspberry Pis. I had no real purpose for doing this, but figured it would be a good learning experience and would leave me with a somewhat realistic environment.

There are already a ton of great resources that cover various aspects of running Kubernetes on Raspberry Pi. This post is just a summary of the steps that worked for me for reference.

# Hardware

I decided a three node cluster would be enough to get started. Here are the items I purchased:

* 3 x [Raspberry Pi 4 Model B (8GB)](https://thepihut.com/collections/raspberry-pi/products/raspberry-pi-4-model-b?variant=31994565689406)
* 3 x [SanDisk MicroSD Card (32GB)](https://thepihut.com/collections/raspberry-pi-sd-cards/products/sandisk-microsd-card-class-10-a1?variant=39641172377795)
* 3 x [Official US Raspberry Pi 4 Power Supply (5.1V 3A)](https://thepihut.com/collections/raspberry-pi-power-supplies/products/raspberry-pi-psu-us)
* 2 x [Cluster Case for Raspberry Pi (with Fans)](https://thepihut.com/collections/raspberry-pi-cases/products/cluster-case-for-raspberry-pi)

Most likely the 4GB model would have been more than enough, but for $20 more you can double the RAM to 8GB. Also, I already had an 8-port switch with some ethernet cables, although each Pi also has on-board Wifi as an option.

The instructions for setting up the cluster case are [available online](https://thepihut.com/blogs/raspberry-pi-tutorials/cluster-case-assembly-instructions). I think it could have been made a little simpler, but overall it wasn't too hard. Pay close attention to the orientation of everything in the pictures.

Here is what my setup looked like after putting it together:
<img src="{{ BASE_PATH }}/assets/images/k8s-rpi/00-rpi-cluster.jpg" class="img-responsive">

# Raspberry Pi Setup

Before we can prepare the SD cards, we have to decide which operating system to use, which then leads to thinking about which Kubernetes distribution to use.

After doing some research, it seemed like there were two main options:

* `Raspberry Pi OS 64-bit` + `K3s`
* `Ubuntu 20.04` + `MicroK8s`

Since Raspberry Pi OS is the official operating system, I decided to go with that and give K3s a try.

### Image SD Cards

1. Download the latest [64-bit version of Raspberry Pi OS](https://downloads.raspberrypi.org/raspios_arm64/images/raspios_arm64-2021-04-09/2021-03-04-raspios-buster-arm64.zip)

2. Use the [Raspberry Pi Imager](https://www.raspberrypi.org/software/) to image the SD cards
* Under *Operating System*, choose *Use Custom*
* Select the OS image downloaded in step 1
* Under *Storage*, select the SD card
* Customize settings by pressing `CMD+SHIFT+X` on Mac
* Set a hostname like `rpi-1`, enable password-based SSH
* Select *Write* to begin imaging
* Repeat the process for each SD card

3. Insert the SD cards into the Pis

4. Connect Pis to the switch with ethernet cables

5. Connect one ethernet cable from the switch to your home router

6. Connect a power supply to each Pi

7. Plug the power supplies into outlets

At this point, the Pis should boot up and get assigned a dynamic IP address on your home network, just like any other device. You can use your router's admin console to determine the IP addresses.

Once you have the IP addresses, you can test SSH to each one by running `ssh pi@<ip-address>`, using the password you set when imaging the SD cards.

### Assign Static IP Addresses

My router had a DHCP range of `192.168.1.2` - `192.168.1.255`, so I adjusted the ending value to `192.168.1.235` in order to reserve some addresses for static assignment. That means we can pick three consecutive addresses somewhere between `192.168.1.236` - `192.168.1.255`.

For this example, we'll use:
```
192.168.1.244
192.168.1.245
192.168.1.246
```

To set the static addresses, do the following:

1. SSH to the first Pi

        ssh pi@<current-ip-address>

2. Install `vim`

        sudo apt-get install vim

3. Edit `/etc/dhcpcd.conf`

        sudo vim /etc/dhcpcd.conf

4. Find the section for static address, uncomment and edit:

        interface eth0
        static ip_address=192.168.1.244/24
        static routers=192.168.1.1
        static domain_name_servers=192.168.1.1

    Replace `192.168.1.1` with your router's IP address.

    Replace `192.168.1.244` with the static address for the first Pi.

5. Reboot the Pi

        sudo reboot

6. After waiting a minute or two, SSH with the static IP

        ssh pi@192.168.1.244

7. Repeat the steps for each Pi.

8. Setup `/etc/hosts` mappings on local machine

        192.168.1.244 rpi-1
        192.168.1.245 rpi-2
        192.168.1.246 rpi-3

9. Create aliases in `~/.zshrc` for easy SSH access

        alias sshrp1='ssh pi@rpi-1'
        alias sshrp2='ssh pi@rpi-2'
        alias sshrp3='ssh pi@rpi-3'

At this point you can test the SSH aliases to confirm that you can SSH to each Pi using a hostname mapped to the static IP address. For more detailed information, see [Setting a Static IP Address on Raspberry Pi](https://www.linuxscrew.com/raspberry-pi-static-ip).

### Setup Password-less SSH

This is not really a requirement to run Kubernetes, but it will be more convenient to not type passwords over and over, plus it will allow us to use Ansible to issue commands to the Raspberry Pi nodes.

1. Generate an SSH key on your local machine

        ssh-keygen

    Accept all the defaults and don't enter a password.

    This will produce `~/.ssh/id_rsa` (private key) and `~/.ssh/id_rsa.pub` (public key)

4. Copy the content of `~/.ssh/id_rsa.pub`

5. SSH to `rpi-1` and run:

        mkdir ~/.ssh
        touch ~/.ssh/authorized_keys
        chmod 0700 ~/.ssh
        chmod 0600 ~/.ssh/authorized_keys
        vi ~/.ssh/authorized_keys

    Paste contents of the copied `id_rsa.pub` and save the file.

6. Repeat the process for each Pi

At this point, you should be able to run any of the SSH aliases without entering a password.

I also followed a similar process to setup password-less SSH between all of the Raspberry Pi nodes, but this is also just a nice-to-have and not necessary.

# Install k3s

For installing k3s, I used the [k3s-ansible](https://github.com/k3s-io/k3s-ansible) setup, which required that I get Ansible on my laptop. I already had `pip3` installed through `homebrew`, so installing Ansible amounted to:

```
pip3 install ansible --user
export PATH="/Users/bbende/Library/Python/3.7/bin:$PATH"
```

Now for running k3s-ansible...

1. Clone the [k3s-ansible](https://github.com/k3s-io/k3s-ansible) repo

2. Create the inventory by copying the example:

        cp -R inventory/sample inventory/rpi

3. Edit `inventory/rpi/hosts.ini` to look like the following:

        [master]
        192.168.1.244

        [node]
        192.168.1.245
        192.168.1.246

        [k3s_cluster:children]
        master
        node

4. Edit `inventory/rpi/group_vars/all.yml` to set the k3s_version:

        k3s_version: v1.21.0+k3s1

5. Launch the setup, this will take a few minutes:

        ansible-playbook site.yml -i inventory/rpi/hosts.ini


### Configure kubectl

Assuming the setup completed successfully, you can configure `kubectl` on our local machine to connect to the k3s cluster.

1. Transfer the Kubeconfig from the master node to your local machine

        scp pi@192.168.1.244:~/.kube/config ~/.kube/config-rpi-k3s

2. Configure `KUBECONFIG` environment variable in `.zshrc`

        export KUBECONFIG="~/.kube/config"
        export KUBECONFIG="$KUBECONFIG:~/.kube/config-rpi-k3s"

3. Use `kubectl` to check the available config contexts

        kubectl config get-contexts

        CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
        *         docker-desktop   docker-desktop   docker-desktop
                  minikube         minikube         minikube
                  rpi-k3s          default          default

    If the name of the Raspberry Pi context is something different, you can rename it using the command:

        kubectl config rename-context <CURRENT_NAME> <NEW_NAME>

4. Use `kubectl` to view the nodes of the `rpi-k3s` cluster

        kubectl --context rpi-k3s get nodes

        NAME    STATUS   ROLES                  AGE   VERSION
        rpi-3   Ready    <none>                 9d    v1.21.0+k3s1
        rpi-2   Ready    <none>                 9d    v1.21.0+k3s1
        rpi-1   Ready    control-plane,master   9d    v1.21.0+k3s1

At this point you should have a working k3s cluster to play around with.

As a next step, you can try following the k3s docs for [installing the Kubernetes Dashboard](https://rancher.com/docs/k3s/latest/en/installation/kube-dashboard/).
