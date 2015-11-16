# Introduction
This document describes how I set up the UltraGrid environment on CloudSigma.

# Virtual Machines
I created three virtual machines using the Fedora 22 (Server) image. I 
installed all updates on it, connected them to a "private network" and gave
them all in addition to a static public IP address a static RFC 1918 address on
the `eth1` interface in the `10.10.10.0/24` range. I set the static IP 
addresses and MTU to 9000 bytes using `nmtui` on this interface.

Make sure the virtual machines are in the same infrastructure as to get 
optimal network performance over the private network (the AGI column in the 
CloudSigma dashboard). You may need to stop all servers and start them again.

## Packages
We install some basic packages:

    $ sudo dnf -y install fedora-packager make autoconf automake gcc \
        gcc-c++ NetworkManager-tui vim ImageMagick ImageMagick-devel iftop \
	opencv-devel

Also install RPM Fusion repository (for ffmpeg/libavcodec support):

    $ sudo dnf -y install http://download1.rpmfusion.org/free/fedora/rpmfusion-free-release-22.noarch.rpm

Now, install the dependencies from RPM Fusion:

    $ sudo dnf -y install ffmpeg-devel

Disable the firewall:

    $ sudo systemctl stop firewalld
    $ sudo systemctl disable firewalld

For some reason, we need to reboot to get rid of the firewall completely.

### Ubuntu with GL

    $ sudo apt-get install build-essential git automake autoconf mesa-common-dev libglew-dev freeglut3-dev libsdl1.2-dev zip

## Network Buffers
We need to modify the network buffers (on all machines). Modify
`/etc/sysctl.conf` and add the following:

    net.core.rmem_max=18247680

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

# Performance Testing
For uncompressed 4K video at 24fps we need about 3.6GBit/s. So we need to be
able to get up to this speed over the private network. See 
[this](https://www.sitola.cz/igrid/index.php/UltraGrid_FAQ#Estimating_available_bandwidth) 
wiki page for downloading the modified `iperf`.

On the `sender`:

    $ ./iperf-2.0.4/src/iperf -u -s -l 8500 -i 1 -w 16M

On the `transceiver`:

    $ ./iperf-2.0.4/src/iperf -u -c 10.10.10.1 -l 8500 -i 1 -w 16M -b 4G -d

Not that we use `4G` here as we want to be able to send roughly 4GBit/s.

It doesn't seem to get reliable 4G though, and also have some loss:

    ------------------------------------------------------------
    Server listening on UDP port 5001
    Receiving 8500 byte datagrams
    UDP buffer size: 32.0 MByte (WARNING: requested 16.0 MByte)
    ------------------------------------------------------------
    ------------------------------------------------------------
    Client connecting to 10.10.10.1, UDP port 5001
    Sending 8500 byte datagrams
    UDP buffer size: 16.0 MByte
    ------------------------------------------------------------
    [  4] local 10.10.10.2 port 57579 connected with 10.10.10.1 port 5001
    [  3] local 10.10.10.2 port 5001 connected with 10.10.10.1 port 59603
    [ ID] Interval       Transfer     Bandwidth       Jitter   Lost/Total Datagrams
    [  3]  0.0- 1.0 sec    432 MBytes  3.63 Gbits/sec  0.009 ms  610/53928 (1.1%)
    [ ID] Interval       Transfer     Bandwidth
    [  4]  0.0- 1.0 sec    282 MBytes  2.37 Gbits/sec
    [ ID] Interval       Transfer     Bandwidth       Jitter   Lost/Total Datagrams
    [  3]  1.0- 2.0 sec    407 MBytes  3.42 Gbits/sec  0.008 ms  381/50640 (0.75%)
    [ ID] Interval       Transfer     Bandwidth
    [  4]  1.0- 2.0 sec    394 MBytes  3.31 Gbits/sec
    [ ID] Interval       Transfer     Bandwidth       Jitter   Lost/Total Datagrams
    [  3]  2.0- 3.0 sec    381 MBytes  3.20 Gbits/sec  0.010 ms 4919/51908 (9.5%)
    [ ID] Interval       Transfer     Bandwidth
    [  4]  2.0- 3.0 sec    368 MBytes  3.08 Gbits/sec
    [ ID] Interval       Transfer     Bandwidth
    [  4]  3.0- 4.0 sec    428 MBytes  3.59 Gbits/sec
    [ ID] Interval       Transfer     Bandwidth       Jitter   Lost/Total Datagrams
    [  3]  3.0- 4.0 sec    412 MBytes  3.46 Gbits/sec  0.006 ms 1052/51917 (2%)
    [ ID] Interval       Transfer     Bandwidth       Jitter   Lost/Total Datagrams
    [  3]  4.0- 5.0 sec    411 MBytes  3.45 Gbits/sec  0.008 ms  890/51582 (1.7%)
    [ ID] Interval       Transfer     Bandwidth
    [  4]  4.0- 5.0 sec    397 MBytes  3.33 Gbits/sec
    [ ID] Interval       Transfer     Bandwidth       Jitter   Lost/Total Datagrams
    [  3]  5.0- 6.0 sec    403 MBytes  3.38 Gbits/sec  0.009 ms  476/50239 (0.95%)
    [ ID] Interval       Transfer     Bandwidth
    [  4]  5.0- 6.0 sec    424 MBytes  3.55 Gbits/sec
    [ ID] Interval       Transfer     Bandwidth       Jitter   Lost/Total Datagrams
    [  3]  6.0- 7.0 sec    388 MBytes  3.26 Gbits/sec  0.003 ms  556/48469 (1.1%)
    [ ID] Interval       Transfer     Bandwidth
    [  4]  6.0- 7.0 sec    380 MBytes  3.19 Gbits/sec
    [ ID] Interval       Transfer     Bandwidth       Jitter   Lost/Total Datagrams
    [  3]  7.0- 8.0 sec    384 MBytes  3.22 Gbits/sec  0.010 ms 1902/49222 (3.9%)
    [ ID] Interval       Transfer     Bandwidth
    [  4]  7.0- 8.0 sec    348 MBytes  2.92 Gbits/sec
    [ ID] Interval       Transfer     Bandwidth       Jitter   Lost/Total Datagrams
    [  3]  8.0- 9.0 sec    419 MBytes  3.52 Gbits/sec  0.003 ms  210/51906 (0.4%)
    [ ID] Interval       Transfer     Bandwidth
    [  4]  8.0- 9.0 sec    407 MBytes  3.41 Gbits/sec
    [ ID] Interval       Transfer     Bandwidth       Jitter   Lost/Total Datagrams
    [  3]  9.0-10.0 sec    409 MBytes  3.43 Gbits/sec  0.009 ms 1455/51927 (2.8%)
    [ ID] Interval       Transfer     Bandwidth
    [  4]  9.0-10.0 sec    383 MBytes  3.21 Gbits/sec
    [ ID] Interval       Transfer     Bandwidth
    [  4]  0.0-10.0 sec  3.72 GBytes  3.18 Gbits/sec
    [  4] Sent 470046 datagrams
    [ ID] Interval       Transfer     Bandwidth       Jitter   Lost/Total Datagrams
    [  3]  0.0-10.0 sec  3.95 GBytes  3.39 Gbits/sec  5.006 ms 12450/511740 (2.4%)
    [  3]  0.0-10.0 sec  1 datagrams received out-of-order
    [  4] Server Report:
    [ ID] Interval       Transfer     Bandwidth       Jitter   Lost/Total Datagrams
    [  4]  0.0-10.0 sec  3.62 GBytes  3.10 Gbits/sec  2.130 ms 12378/470045 (2.6%)
    [  4]  0.0-10.0 sec  1 datagrams received out-of-order

# UltraGrid
Clone the repository:

    $ git clone http://seth.ics.muni.cz/git/ultragrid.git
    $ git checkout test-no-ssrc-change

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

    TARGET=10.10.10.2

    #MTU=1500
    MTU=8500

    #LIMIT=auto
    LIMIT=unlimited

    $UV -c $COMPRESSION -t $CAPTURE $TARGET -P 5006 -m $MTU -l $LIMIT

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

    FILTER=
    #FILTER="--capture-filter logo:surfnet.pam:0:0"
    #FILTER="--capture-filter text:foo"
    #FILTER="--capture-filter blank:50:50:500:500:black"
    #FILTER="--capture-filter grayscale"
    #FILTER="--capture-filter flip"
    #FILTER="--capture-filter mirror"

    $HD_RUM $FILTER 32M 5006 -c $COMPRESSION -m $MTU -l $LIMIT -P 5006 $TARGET

## Receiver

    #!/bin/sh
    UV=$HOME/ultragrid/bin/uv

    #MTU=1500
    MTU=8500

    DISP=dummy
    #DISP=gl

    $UV -d $DISP -m $MTU -P 5006

# Logo
The transceiver can add a logo to the image. This needs to be in PAM format. I 
took the logo from 
`https://www.surf.nl/binaries/werkmaatschappijlogo/content/gallery/surf/logos/surfnet.png`.

Now you can convert it to PAM:

    $ convert surfnet.png surfnet.pam

# Running
## Sender

    UltraGrid 1.3 (rev v1.3-300-ge7cdd9d)
    Display device   : none
    Capture device   : testcard
    Audio capture    : none
    Audio playback   : none
    MTU              : 8500 B
    Video compression: none
    Audio codec      : PCM
    Network protocol : UltraGrid RTP
    Audio FEC        : none
    Video FEC        : none

    Display initialized-none

    ...

    Testcard capture set to 3840x2160, bpp 2.000000
    Video capture initialized-testcard
    [testcard] 1 frames in 81.4886 seconds = 0.0122717 FPS
    [testcard] 118 frames in 5.00108 seconds = 23.5949 FPS
    [testcard] 120 frames in 5.0081 seconds = 23.9612 FPS
    [testcard] 117 frames in 5.03239 seconds = 23.2494 FPS
    [testcard] 119 frames in 5.00337 seconds = 23.784 FPS
    [testcard] 120 frames in 5.02423 seconds = 23.8843 FPS
    [testcard] 119 frames in 5.0068 seconds = 23.7677 FPS
    [testcard] 120 frames in 5.0098 seconds = 23.9531 FPS
    [testcard] 119 frames in 5.02256 seconds = 23.6931 FPS
    [testcard] 118 frames in 5.01106 seconds = 23.5479 FPS
    [testcard] 120 frames in 5.00804 seconds = 23.9615 FPS

    ...

## Transceiver

    using UDP send and receive buffer size of 134217728 bytes
    initializing packet queue for 16777 items
    Cannot set recv buffer!
    listening on *:5006
    Cannot set send buffer!
    New incoming video format detected: 3840x2160 @23.98p, codec UYVY
    Received 378095592 bytes in 5.00054 seconds = 75610955 B/s.
    [708b4c8c->10.10.10.3:5006] 26 frames in 5.01822 seconds = 5.18112 FPS
    SSRC 71c26091: 222840 packets expected, 165012 was received (74.05%).
    Received 1873390296 bytes in 5 seconds = 374677898 B/s.
    [708b4c8c->10.10.10.3:5006] 117 frames in 5.01381 seconds = 23.3356 FPS
    SSRC 71c26091: 239250 packets expected, 210009 was received (87.78%).
    Received 1840277796 bytes in 5.02151 seconds = 366478679 B/s.
    [708b4c8c->10.10.10.3:5006] 116 frames in 5.06051 seconds = 22.9226 FPS
    SSRC 71c26091: 235020 packets expected, 210657 was received (89.63%).
    Received 1761761472 bytes in 5.00003 seconds = 352349891 B/s.
    [708b4c8c->10.10.10.3:5006] 111 frames in 5.00944 seconds = 22.1581 FPS
    SSRC 71c26091: 231080 packets expected, 189048 was received (81.81%).
    Received 1759322268 bytes in 5.00367 seconds = 351606309 B/s.
    
    ...

## Receiver

    UltraGrid 1.3 (rev v1.3-300-ge7cdd9d)
    Display device   : dummy
    Capture device   : none
    Audio capture    : none
    Audio playback   : none
    MTU              : 8500 B
    Video compression: none
    Audio codec      : PCM
    Network protocol : UltraGrid RTP
    Audio FEC        : none
    Video FEC        : none

    Display initialized-dummy
    Video capture initialized-none
    SSRC 708b4c8c: 10 packets expected, 10 was received (100.00%).
    New incoming video format detected: 3840x2160 @23.98p, codec UYVY
    SSRC 708b4c8c: 221590 packets expected, 209051 was received (94.34%).
    [dummy] 113 frames in 5.03811 seconds = 22.429 FPS
    SSRC 708b4c8c: 203420 packets expected, 199877 was received (98.26%).
    [dummy] 101 frames in 5.03989 seconds = 20.0401 FPS
    SSRC 708b4c8c: 221200 packets expected, 213391 was received (96.47%).
    [dummy] 111 frames in 5.0225 seconds = 22.1005 FPS
    SSRC 708b4c8c: 219230 packets expected, 211008 was received (96.25%).
    [dummy] 106 frames in 5.03552 seconds = 21.0505 FPS
    [dummy] 105 frames in 5.00071 seconds = 20.997 FPS
    SSRC 708b4c8c: 209350 packets expected, 205871 was received (98.34%).
    Video decoder statistics: 609 total: 599 displayed / 0 dropped / 73 corrupted / 10 missing frames.
    [dummy] 110 frames in 5.02182 seconds = 21.9044 FPS
    SSRC 708b4c8c: 219220 packets expected, 213831 was received (97.54%).
    [dummy] 109 frames in 5.02464 seconds = 21.6931 FPS
    SSRC 708b4c8c: 219230 packets expected, 212853 was received (97.09%).
    [dummy] 108 frames in 5.02293 seconds = 21.5014 FPS
    SSRC 708b4c8c: 219220 packets expected, 214994 was received (98.07%).
    [dummy] 109 frames in 5.03813 seconds = 21.635 FPS
    SSRC 708b4c8c: 223180 packets expected, 213944 was received (95.86%).

    UltraGrid has crashed (Segmentation fault).

    Please send a bug report to address ultragrid-dev@cesnet.cz.
    You may find some tips how to report bugs in file REPORTING-BUGS distributed with UltraGrid.
    receiver.sh: line 7:   881 Segmentation fault      (core dumped) $UV -d dummy -m $MTU -P 5006

    ...

# Coredump
Put:

    *          soft    core      unlimited

In `/etc/security/limits.d/30-core.conf`.

Or: 

    $ ulimit -c unlimited
    $ ls /var/lib/systemd/coredump

# NAT rules
On all the transceivers:

    #!/bin/sh

    sudo iptables -F
    sudo iptables -F -t nat

    sudo sysctl -w net.ipv4.conf.all.route_localnet=1

    # enable DNAT
    sudo iptables -t nat -A PREROUTING -p udp --dport 5006 -d 10.20.30.74 -j DNAT --to 127.0.0.1:5006

    # set static ARP
    sudo arp -s 10.20.30.74 00:60:dd:46:13:ae

    sudo conntrack --flush


