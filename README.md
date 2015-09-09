# Introduction
This document describes how I set up the UltraGrid environment on CloudSigma.

# Virtual Machines
I created three virtual machines using the Fedora 22 (Server) image. I 
installed all updates on it, connected them to a "private network" and gave
them all in addition to a static public IP address a static RFC 1918 address on
the `eth1` interface in the `10.10.10.0/24` range. I set the static IP 
addresses and MTU to 9000 bytes using `nmtui` on this interface.

## Packages
We install some basic packages:

    $ sudo dnf -y install fedora-packager make autoconf automake gcc \
        gcc-c++ NetworkManager-tui vim

Also install RPM Fusion repository (for ffmpeg/libavcodec support):

    $ sudo dnf -y install http://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-22.noarch.rpm

Now, install the dependencies from RPM Fusion:

    $ sudo dnf -y install ffmpeg-devel

Disable the firewall:

    $ sudo systemctl stop firewalld
    $ sudo systemctl disable firewalld

For some reason, we need to reboot to get rid of the firewall completely.

## Network Buffers
We need to modify the network buffers (on all machines). Modify 
`/etc/sysctl.conf` and add the following:

    # Extended network buffers for UltraGrid
    net.core.wmem_max = 33554432
    net.core.wmem_default = 33554432
    net.core.rmem_max = 33554432
    net.core.rmem_default = 33554432

Now to activate:

    $ sudo sysctl -p

## Ping
In order to test if the MTU of 9000 is working properly, there is a simple
test to perform:

    $ ping -M do -s 8500 -c 4 10.10.10.1
    PING 10.10.10.1 (10.10.10.1) 8500(8528) bytes of data.
    8508 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=0.195 ms
    8508 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=0.240 ms
    8508 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=0.240 ms
    8508 bytes from 10.10.10.1: icmp_seq=4 ttl=64 time=0.233 ms

    --- 10.10.10.1 ping statistics ---
    4 packets transmitted, 4 received, 0% packet loss, time 2999ms
    rtt min/avg/max/mdev = 0.195/0.227/0.240/0.018 ms

This should work between the 3 machines in the private network!

## Roles
The three VMs have the following roles:

    10.10.10.1  sender
    10.10.10.2  transceiver
    10.10.10.3  receiver

# UltraGrid
I used the `git` version of UltraGrid from 
`http://seth.ics.muni.cz/git/ultragrid.git` (the `master` branch).

To compile UltraGrid, use the following:

    $ ./autogen.sh && ./configure && make

# Scripts
There are three scripts, one for the sender, the transceiver and the receiver.

## Sender

    #!/bin/sh
    UV=$HOME/ultragrid/bin/uv

    #CAPTURE=testcard:1920:1080:23.98:UYVY	# full HD
    #CAPTURE=testcard:1920:1080:60:UYVY   # full HD higher frame rate to test MTU
    CAPTURE=testcard:3840:2160:23.98:UYVY	# 4K
    #CAPTURE=testcard:3840:2160:60:UYVY	# 4K higher frame rate to test MTU

    COMPRESSION=none
    #COMPRESSION=libavcodec:codec=MJPEG
    #COMPRESSION=libavcodec:codec=H.264

    TARGET=10.10.10.1

    #MTU=1500
    MTU=8500

    #LIMIT=auto
    LIMIT=unlimited

    $UV -c $COMPRESSION -t $CAPTURE $TARGET -P 5006 -m $MTU -l $LIMIT

## Receiver

    #!/bin/sh
    UV=$HOME/ultragrid/bin/uv

    #MTU=1500
    MTU=8500

    $UV -d dummy -m $MTU -P 5006

## Transceiver

    #!/bin/sh
    HD_RUM=$HOME/ultragrid/bin/hd-rum-transcode

    COMPRESSION=none
    #COMPRESSION=libavcodec:codec=MJPEG
    #COMPRESSION=libavcodec:codec=H.264

    TARGET=10.10.10.3
    
    #MTU=1500
    MTU=8500

    #LIMIT=auto
    LIMIT=unlimited

    #FILTER=
    FILTER="--capture-filter logo:surfnet.pam:0:0"

    $HD_RUM $FILTER 32M 5006 -c $COMPRESSION -m $MTU -l $LIMIT -P 5006 $TARGET

# Running

