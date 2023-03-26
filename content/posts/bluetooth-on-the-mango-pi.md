---
title: "Setting up Bluetooth on the MangoPi MQ-Pro, and testing it out with a Bluetooth access point"
date: 2023-03-25T17:17:55+01:00
tags:
  - mangopi-mq-pro
  - risc-v
  - bluetooth
  - ubuntu
draft: false
---
I received my MangoPi MQ-Pro a few months ago, and was very eager to test out
all the hardware which I use the most on SBC's. Bluetooth is not one of the
features I use a lot on SBC's. But after having tested out a lot of operating
systems and hardware features of the board, I decided to also try and see what
the status of Bluetooth on the board was.

Because I use my MangoPi headless almost always, I couldn't test Bluetooth with
an input device. I also don't use audio on the board. So I decided to try to
set up a Bluetooth access point on the board. I already successfully tried this
on a Raspberry Pi.

I used the [Ubuntu 22.10 RISC-V image for the SiPeed LicheeRV Dock](https://ubuntu.com/download/risc-v).
This board is very similar to the MangoPi MQ-Pro (same CPU, WiFi+BT radio...).
The first important thing to check is you're running kernel `5.19.0-1007-allwinner`
or higher. If that's not the case, update the system:
```
$ sudo apt udpate
$ sudo apt upgrade
$ sudo reboot
```
After this, when trying to load the kernel module necessary for Bluetooth, a
warning in the kernel logs will appear that the firmware and config for the
radio are not available.
```
$ sudo modprobe hci_uart
$ sudo dmesg | grep hci0
...
[ 9197.854777] Bluetooth: hci0: RTL: loading rtl_bt/rtl8723ds_fw.bin
[ 9197.855274] bluetooth hci0: Direct firmware load for rtl_bt/rtl8723ds_fw.bin failed with error -2
[ 9197.855319] Bluetooth: hci0: RTL: firmware file rtl_bt/rtl8723ds_fw.bin not found
...
```
So it seems the firmware package of Ubuntu doesn't include the necessary firmware.
No problem, the [Armbian firmware](https://github.com/armbian/firmware) does
seem to include these. We can then fetch the firmware files from these repositories:
```
sudo wget -O /lib/firmware/rtl_bt/rtl8723ds_config.bin https://github.com/armbian/firmware/raw/master/rtl_bt/rtl8723ds_config.bin
sudo wget -O /lib/firmware/rtl_bt/rtl8723ds_fw.bin https://github.com/armbian/firmware/raw/master/rtl_bt/rtl8723ds_fw.bin
```
If we reload the `hci_uart` module after this, the firmware files get picked up,
and the radio works:
```
$ sudo modprobe -r hci_uart
$ sudo modprobe hci_uart
$ sudo dmesg | grep hci0
...
[ 9399.931540] Bluetooth: hci0: RTL: loading rtl_bt/rtl8723ds_fw.bin
[ 9399.932142] Bluetooth: hci0: RTL: loading rtl_bt/rtl8723ds_config.bin
...
$ hciconfig # only works after installing bluez-tools
hci0:   Type: Primary  Bus: UART
        BD Address: 68:B9:D3:7C:AD:57  ACL MTU: 1021:8  SCO MTU: 255:12
        UP RUNNING PSCAN 
        RX bytes:3624 acl:0 sco:0 events:168 errors:0
        TX bytes:34947 acl:0 sco:0 commands:167 errors:0
```

Now that the radio is up and running, we can prepare the userspace for Bluetooth.
First of all we will need to install `bluez` and `bluez-utils`, and enable the
Bluetooth service:
```
sudo apt install bluez bluez-tools
sudo apt install bluez bluez-tools
```

There you have it, you can start using Bluetooth with `bluetoothctl`. If you
also want to configure a Bluetooth access point, read on!

A few new config files will need to be created for this to work, so keep your
editor of choice by hand.

Create the `/etc/systemd/network/pan0.netdev`-file with the following content:
```
[NetDev]
Name=pan0
Kind=bridge
```

Do the same for the `/etc/systemd/network/pan0.network`-file with the following
content:
```
[Match]
Name=pan0

[Network]
Address=172.20.1.1/24
DHCPServer=yes

[DHCPServer]
DNS=<DNS server(s)>
```
You will need to replace the DNS server(s) with one or more (space separated)
DNS servers.

`/etc/systemd/system/bt-agent.service` needs the following content:
```
[Unit]
Description=Bluetooth Auth Agent

[Service]
ExecStart=/usr/bin/bt-agent -c NoInputNoOutput
Type=simple

[Install]
WantedBy=multi-user.target
```

`/etc/systemd/system/bt-network.service` needs to look as follows:
```
[Unit]
Description=Bluetooth NEP PAN
After=pan0.network

[Service]
ExecStart=/usr/bin/bt-network -s nap pan0
Type=simple

[Install]
WantedBy=multi-user.target
```

Enable ipv4 forwarding by creating a `/etc/sysctl.d/bt-pan.conf`-file with the
following content:
```
# Enable IPv4 routing
net.ipv4.ip_forward=1
```

Enable ip tables rules:
```
sudo apt install -y netfilter-persistent iptables-persistent
sudo iptables -t nat -A POSTROUTING -o <outgoing interface> -j MASQUERADE
sudo netfilter-persistent save
```

You should replace the `outgoing interface` with the network interface you are
currently using, for example `wlan0` for WiFi, `en.....` for a USB to LAN adapter.

Then start and enable everything:
```
sudo systemctl enable --now systemd-networkd
sudo systemctl enable --now bt-agent
sudo systemctl enable --now bt-network
```

You can then connect new devices by issuing the following command:
```
sudo bt-adapter --set Discoverable 1
```

Go into the Bluetooth settings of your phone/tablet/laptop/... to search for a
new Bluetooth devices, and it should connect to it and use it as a Bluetooth
access point. After this is done, you can turn of the discovery:
```
sudo bt-adapter --set Discoverable 0
```

When executing some tests using iperf3, with my Pixel 6 as a client, the resulst
seem to be pretty good for what you can expect from a 6 year old Bluetooth 4.2
module, the connection also seems to be very stable:
```
-----------------------------------------------------------
Server listening on 5201 (test #1)
-----------------------------------------------------------
Accepted connection from 172.20.1.184, port 57282
[  5] local 172.20.1.1 port 5201 connected to 172.20.1.184 port 57284
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.00   sec  65.0 KBytes   533 Kbits/sec                  
[  5]   1.00-2.00   sec  82.0 KBytes   672 Kbits/sec                  
[  5]   2.00-3.00   sec  87.7 KBytes   718 Kbits/sec                  
[  5]   3.00-4.00   sec  66.5 KBytes   544 Kbits/sec                  
[  5]   4.00-5.00   sec  83.4 KBytes   683 Kbits/sec                  
[  5]   5.00-6.00   sec  87.7 KBytes   718 Kbits/sec                  
[  5]   6.00-7.00   sec  69.4 KBytes   568 Kbits/sec                  
[  5]   7.00-8.00   sec  90.5 KBytes   741 Kbits/sec                  
[  5]   8.00-9.00   sec  75.7 KBytes   620 Kbits/sec                  
[  5]   9.00-10.00  sec  90.5 KBytes   741 Kbits/sec                  
[  5]  10.00-11.00  sec  89.1 KBytes   730 Kbits/sec                  
[  5]  11.00-11.21  sec  8.54 KBytes   337 Kbits/sec                  
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-11.21  sec   896 KBytes   655 Kbits/sec                  receiver
-----------------------------------------------------------
Server listening on 5201 (test #2)
-----------------------------------------------------------
Accepted connection from 172.20.1.184, port 57286
[  5] local 172.20.1.1 port 5201 connected to 172.20.1.184 port 57288
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec   239 KBytes  1.96 Mbits/sec    0   35.4 KBytes       
[  5]   1.00-2.00   sec  76.4 KBytes   625 Kbits/sec    0   39.6 KBytes       
[  5]   2.00-3.00   sec   102 KBytes   834 Kbits/sec    0   43.8 KBytes       
[  5]   3.00-4.00   sec   102 KBytes   834 Kbits/sec    0   55.1 KBytes       
[  5]   4.00-5.00   sec   127 KBytes  1.04 Mbits/sec    0   65.0 KBytes       
[  5]   5.00-6.00   sec   127 KBytes  1.04 Mbits/sec    0   65.0 KBytes       
[  5]   6.00-7.00   sec  0.00 Bytes  0.00 bits/sec    0   65.0 KBytes       
[  5]   7.00-8.00   sec   127 KBytes  1.04 Mbits/sec    0   65.0 KBytes       
[  5]   8.00-9.00   sec   127 KBytes  1.04 Mbits/sec    0   65.0 KBytes       
[  5]   9.00-10.00  sec  0.00 Bytes  0.00 bits/sec    0   65.0 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.05  sec  1.00 MBytes   838 Kbits/sec    0             sender
```

After this experiment I also tried to enable Bluetooth support in the excellent
[Arch image](https://github.com/sehraf/d1-riscv-arch-image-builder) for the D1.
I didn't succeed in this yet. I am probably missing some kernel config options.
