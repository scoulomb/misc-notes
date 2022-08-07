## WOL

**Objective**: Turn on the NAS remotely from LAN or WAN.


### Get MAC @dresss of NAS

Need to find MAC when NAS is turned on ! 

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ arp -a local.nas.coulombel.net
nas (192.168.1.88) at 24:5e:be:3f:2b:f7 [ether] on wlo1
````

or via router web UI (`192.168.1.1`).


### Shut down the NAS

Then shut down via
- NAS UI (or scheduled power off) 
- QNAP manager Android app 
- button

or ssh (seems does not disable WOL) 

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ ssh admin@192.168.1.88 /sbin/poweroff #(-f) most of tests done with -f but avoid
admin@192.168.1.88's password: # could use ssh key here
````
If we open UI before SSH will see service shut down progress

Power cut could cause issue, https://community.synology.com/enu/forum/17/post/21402


### etherwake

https://www.mkssoftware.com/docs/man1/etherwake.1.asp

Then `sudo apt install etherwake`

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo etherwake 24:5e:be:3f:2b:f7 
SIOCGIFHWADDR on eth0 failed: No such device
````

We have to specify the interface!

````
oulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ etherwake -h -u
etherwake: invalid option -- 'h'
usage: etherwake [-i <ifname>] [-p aa:bb:cc:dd[:ee:ff]] 00:11:22:33:44:55

        This program generates and transmits a Wake-On-LAN (WOL)
        "Magic Packet", used for restarting machines that have been
        soft-powered-down (ACPI D3-warm state).
        It currently generates the standard AMD Magic Packet format, with
        an optional password appended.

        The single required parameter is the Ethernet MAC (station) address
        of the machine to wake or a host ID with known NSS 'ethers' entry.
        The MAC address may be found with the 'arp' program while the target
        machine is awake.

        Options:
                -b      Send wake-up packet to the broadcast address.
                -D      Increase the debug level.
                -i ifname       Use interface IFNAME instead of the default 'eth0'.
                -p <pw>         Append the four or six byte password PW to the packet.
                                        A password is only required for a few adapter types.
                                        The password may be specified in ethernet hex format
                                        or dotted decimal (Internet address)
                -p 00:22:44:66:88:aa
                -p 192.168.1.1
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$
````

So 

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo etherwake -i wlo1  24:5e:be:3f:2b:f7 
````

**Warning**: `-p` is for the password and not for the port.

### pywakeonlan

or use Python equivalent https://github.com/remcohaszing/pywakeonlan 

````
pip install wakeonlan
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo wakeonlan 24:5e:be:3f:2b:f7
[sudo] password for scoulomb: 
[Tested OK after ssh shutdown]
````
or via python console 
<!-- tested OK - 2 beeps + UI-->

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo pip install wakeonlan # install as sudo
Collecting wakeonlan
  Downloading wakeonlan-2.1.0-py3-none-any.whl (4.4 kB)
Installing collected packages: wakeonlan
Successfully installed wakeonlan-2.1.0
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo python3
Python 3.9.7 (default, Jun 22 2022, 20:11:26) 
[GCC 11.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> from wakeonlan import send_magic_packet
>>> send_magic_packet('24:5e:be:3f:2b:f7')
>>> 

````
With python I had to send several time command (or was not patient)

More raw solution (not tested):https://stackoverflow.com/questions/31588035/bash-one-line-command-to-send-wake-on-lan-magic-packet-without-specific-tool

### Socket and Python 

Similar to https://github.com/scoulomb/http-over-socket/blob/main/1-client/cli.py

Code from wikipedia: https://en.wikipedia.org/wiki/Wake-on-LAN

````python
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ cat mywol.py
import socket

def wol(lunaMacAddress: bytes):
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    s.setsockopt(socket.SOL_SOCKET, socket.SO_BROADCAST, 1)

    magic = b'\xff' * 6 + lunaMacAddress * 16
    s.sendto(magic, ('<broadcast>', 7))

if __name__ == '__main__':
    # pass to wol the mac address of the ethernet port of the appliance to wakeup
    wol(b'\x24\x5e\xbe\x3f\x2b\xf7')
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ python3 mywol.py
````

It works with both port 7 and port 9.

`'<broadcast>'` matches IP `255.255.255.255`.
This can be verified by doing 

````
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo tcpdump -i wlo1 port 7
[sudo] password for scoulomb:
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on wlo1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
16:55:18.726292 IP scoulomb-HP-Pavilion-TS-Sleekbook-14.54274 > 255.255.255.255.echo: UDP, length 102
````

We can replace in our SFR BOX lan by `192.168.1.255`.

**Notes**:
Note that it matches default value of pywakeonlan: https://github.com/remcohaszing/pywakeonlan#as-a-standalone-script

Is it possible to not use broadcast IP and use directly NAS IP on the LAN which is `192.168.1.88`.

```python
s.sendto(magic, ('192.168.1.88', 9))
```
From test it did not work.

### Comment on Etherwake

#### Etherwake does not use UDP

Our etherwake version does not allow to go through UDP (no IP and Port): Cf `-b` field does not allow to set an IP and port not pressent as option.

I suspect Etherwake use ARP directly. This also explains why: https://doc.ubuntu-fr.org/wakeonlan
> Contrairement à wakeonlan, le paquet etherwake ne permet pas de démarrer un pc en passant par internet. Nous installerons donc wakeonlan disponible dans le dépôt universe. 

Assumption is confirmed in the code https://coral.googlesource.com/busybox/+/refs/heads/release-chef/networking/ether-wake.c#101.
It uses `SOCK_PACKET` or `SOCK_RAW`, see here:https://man7.org/linux/man-pages/man2/socket.2.html
And not `SOCK_DGRAM` (UDP) or `SOCK_STREAM` (TCP): https://stackoverflow.com/questions/5815675/what-is-sock-dgram-and-sock-stream



#### How to not specify mac address with Etherwake 

See https://linux.die.net/man/8/ether-wake

> The single required parameter is a station (MAC) address or a **host ID that can be translated to a MAC address by an ethers(5) database specified in nsswitch.conf(5)**


For this do 


```shell
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo arp -d 192.168.1.88
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ arp -e | grep nas
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ cat /etc/ethers
24:5e:be:3f:2b:f7 192.168.1.88
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ arp -e | head -n 2
Address                  HWtype  HWaddress           Flags Mask            Iface
10.42.0.212              ether   6e:34:d0:f4:94:98   C                     cni0
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ arp -e | grep nas
nas                      ether   24:5e:be:3f:2b:f7   CM                    wlo1
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo etherwake -i wlo1  192.168.1.88
```

<!-- could not set with custom DNS,
Also it shows name "nas" in arp table, no clue how got it
in etherwake command we have to use ip
-->

### How does it work?

#### Reminder on ARP

From https://en.wikipedia.org/wiki/Address_Resolution_Protocol

> Two computers in an office (Computer 1 and Computer 2) are connected to each other in a local area network by Ethernet cables and network switches, with no intervening gateways or routers. Computer 1 has a packet to send to Computer 2. Through DNS, it determines that Computer 2 has the IP address 192.168.0.55.

> To send the message, it also requires Computer 2's MAC address. First, Computer 1 uses a cached ARP table to look up 192.168.0.55 for any existing records of Computer 2's MAC address (00:EB:24:B2:05:AC). If the MAC address is found, it sends an Ethernet frame containing the IP packet onto the link with the destination address 00:EB:24:B2:05:AC. If the cache did not produce a result for 192.168.0.55, Computer 1 has to send a broadcast ARP request message (destination FF:FF:FF:FF:FF:FF MAC address), which is accepted by all computers on the local network, requesting an answer for 192.168.0.55.

> Computer 2 responds with an ARP response message containing its MAC and IP addresses. As part of fielding the request, Computer 2 may insert an entry for Computer 1 into its ARP table for future use.

> Computer 1 receives and caches the response information in its ARP table and can now send the packet.[7]
ARP probe

Then when message exchanged can use TCP/UDP layer  encapsulation over ethernet.
See OSI layer and encapsulation: https://en.wikipedia.org/wiki/Encapsulation_(networking)

#### WOL

WOL works in a similar fashion but does not use ARP.

From: https://en.wikipedia.org/wiki/Wake-on-LAN

> Ethernet connections, including home and work networks, wireless data networks and the Internet itself, are based on frames sent between computers. WoL is implemented using a specially designed frame called a magic packet, which is sent to all computers in a network, among them the computer to be awakened. The magic packet contains the MAC address of the destination computer, an identifying number built into each network interface card ("NIC") or other ethernet device in a computer, that enables it to be uniquely recognized and addressed on a network. Powered-down or turned off computers capable of Wake-on-LAN will contain network devices able to "listen" to incoming packets in low-power mode while the system is powered down. If a magic packet is received that is directed to the device's MAC address, the NIC signals the computer's power supply or motherboard to initiate system wake-up, in the same way that pressing the power button would do.

> **The magic packet is sent on the data link layer (layer 2 in the OSI model) and when sent, is broadcast to all attached devices on a given network, using the network broadcast address; the IP-address (layer 3 in the OSI model) is not used.**

> **Because Wake-on-LAN is built upon broadcast technology, it can generally only be used within the current network subnet. There are some exceptions, though, and Wake-on-LAN can operate across any network in practice, given appropriate configuration and hardware, including remote wake-up across the Internet.**

> In order for Wake-on-LAN to work, parts of the network interface need to stay on. This consumes a small amount of standby power, much less than normal operating power. The link speed is usually reduced to the lowest possible speed to not waste power (e.g. a Gigabit Ethernet NIC maintains only a 10 Mbit/s link). Disabling wake-on-LAN when not needed can very slightly reduce power consumption on computers that are switched off but still plugged into a power socket.[5] The power drain becomes a consideration on battery powered devices such as laptops as this can deplete the battery even when the device is completely shut down.

See also section: "Creating and sending the magic packet"

This why we have to know ethernet address in advance.

Nothe the broadcast made here
- at level 2 (mac): `FF:FF:FF:FF:FF:FF` where payload contain mac@ of device to wake-up
- at level 3 (ip): `192.168.1.255`/`255.255.255.255` (not used in etherwake)

Cf. https://fr.wikipedia.org/wiki/Broadcast_(informatique)

See here: https://en.wikipedia.org/wiki/Routing

It confirms we can apply unicast, multicast, broadcast at L2 or L3. See https://reussirsonccna.fr/unicast-multicast-broadcast-oui-mais-quelle-couche/ (anycast not sure at L2).

### Remote WOL (WOW - Wake On Wan)

See wiki"  "wake on internet"
 
Can use to test: https://www.depicus.com/wake-on-lan/woli

What I did is
- Set static IP@ address for NAS in LAN: http://192.168.1.1/network/dhcp (this is recommended to lose MAC mappingt)
- Activated proxy wake on LAN: http://192.168.1.1/network/nat (DNAT)

From https://la-communaute.sfr.fr/t5/installation-et-param%C3%A9trage/wake-on-lan-sur-nb6/td-p/1868922/page/2, we need

 - L'adresse MAC de votre équipement (pour qu'en recevant le magic packet, il sache que c'est lui qui doit se réveiller. Oui en broadcast, tous les équipements de votre LAN receront ce paquet !)
- L'ip publique de votre box
- Masque : 255.255.255.255
- Port : 9
<!-- I will only try port 9 -->

So I tried by changing Python code to `s.sendto(magic, ('109.29.148.109', 9))`
 
It did not work...
We managed to have it working with 2 following actions:
- Do not target from LAN but outside (use phone wifi hotspot with 4G)
  - Unlike simple NAT, we can not use public IP inside the LAN.
<!-- cf. 5 	Nashttp 	TCP 	Port 	9080 	192.168.1.88 	8080 and access UI via 109.29.148.109:9080 (tested OK), mentioned somewhere else -->
- Modify the sender to send more packets (some are loss)

````python
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ cat mywol2.py 
import socket

def wol(lunaMacAddress: bytes):
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    #s.setsockopt(socket.SOL_SOCKET, socket.SO_BROADCAST, 1)

    magic = b'\xff' * 6 + lunaMacAddress * 16
    s.sendto(magic, ('109.29.148.109', 9))

if __name__ == '__main__':
    # pass to wol the mac address of the ethernet port of the appliance to wakeup
    for x in range(0,10):
       wol(b'\x24\x5e\xbe\x3f\x2b\xf7')
````


`etherwake` will not work here for [reasons explained above](#etherwake-does-not-use-udp).

### Android app

We can use https://play.google.com/store/apps/details?id=com.laruedavid.wakeonlan&hl=en_US&gl=US

#### Setup 1: WOL

```shell
`255.255.255.255` or `192.168.1.255`
`24:5E:BE:3F:2B:F7` 
Port: 9
```

This will work only when phone is connected to Home LAN/WIFI network.
<!-- both IP tested OK, re-tested `192.168.1.255` -->

Note use non broadcast IP (`192.168.1.88`) does not work.
<!-- re-tested OK -->
This is consistent with [socket and python section](#socket-and-python).

#### Setup 2: WOW

##### Setup 2a: WOW

```shell
`109.29.148.109`
`24:5E:BE:3F:2B:F7` 
Port: 9
```

This will work only when phone is NOT connected to Home LAN/WIFI network. For instance use 4G network.

#### Setup 2b: WOW with cloud server

To fix this we can tick option use cloud, because in that request will not be sent from the phone 
So working when not connected to Home LAN WIFI (4G) and also connected to Home LAN/WIFI.

I recommend this setup which works from anywhere.

Note the applications also sends several packets.

This application helps us debugging [WOW above section](#remote-wol-wow---wake-on-wan).

<!-- both tested OK -->

### Show tcp dump 

We will send WOL packet via android app and observe the packet received via a laptop  (tcpdump) in same LAN as NAS.

We can access same info via Wireshark (tested in LAB laptop)

<!--
We will not do a dump inside the NAS
As complex to install tcpdump

We will not try sending packet from LAB laptop and observe packet from another laptop
If observe packet and send packet from LAB laptop we have some noise effect
-->

#### Android WOL

We use setup [setup-1-wol](#setup-1-wol) with `192.168.1.255` broadcast IP.

````shell
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo tcpdump udp port 9
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on wlo1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
15:37:27.878316 IP sylvain-s-S10.57731 > 192.168.1.255.discard: UDP, length 102
15:37:27.880688 IP sylvain-s-S10.57731 > 192.168.1.255.discard: UDP, length 102
15:37:27.885418 IP sylvain-s-S10.57731 > 192.168.1.255.discard: UDP, length 102
15:37:27.887570 IP sylvain-s-S10.57731 > 192.168.1.255.discard: UDP, length 102
15:37:27.889846 IP sylvain-s-S10.57731 > 192.168.1.255.discard: UDP, length 102
15:37:27.891836 IP sylvain-s-S10.57731 > 192.168.1.255.discard: UDP, length 102
15:37:27.893949 IP sylvain-s-S10.57731 > 192.168.1.255.discard: UDP, length 102
15:37:27.896225 IP sylvain-s-S10.57731 > 192.168.1.255.discard: UDP, length 102
15:37:27.898621 IP sylvain-s-S10.57731 > 192.168.1.255.discard: UDP, length 102
15:37:27.900951 IP sylvain-s-S10.57731 > 192.168.1.255.discard: UDP, length 102
^C
10 packets captured
10 packets received by filter
0 packets dropped by kernel
````


#### Android WOW

Android phone in 4G. Not using cloud server.

We use setup [setup-2-wow](#setup-2a-wow).

````shell
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo tcpdump udp port 9
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on wlo1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
15:40:50.938713 IP box.36037 > 192.168.1.255.discard: UDP, length 102
15:40:50.940980 IP box.36037 > 192.168.1.255.discard: UDP, length 102
15:40:50.943379 IP box.36037 > 192.168.1.255.discard: UDP, length 102
15:40:50.948054 IP box.36037 > 192.168.1.255.discard: UDP, length 102
15:40:50.950418 IP box.36037 > 192.168.1.255.discard: UDP, length 102
15:40:50.952635 IP box.36037 > 192.168.1.255.discard: UDP, length 102
15:40:50.954897 IP box.36037 > 192.168.1.255.discard: UDP, length 102
15:40:50.957160 IP box.36037 > 192.168.1.255.discard: UDP, length 102
15:40:50.959396 IP box.36037 > 192.168.1.255.discard: UDP, length 102
15:40:50.963811 IP box.36037 > 192.168.1.255.discard: UDP, length 102
^C
10 packets captured
10 packets received by filter
0 packets dropped by kernel
````

We use setup [setup-2b-wow-with-cloud-server](#setup-2b-wow-with-cloud-server).

Android phone in 4G (or not). Using cloud server.
<!-- tested same output when 4G or Not -->

````shell
scoulomb@scoulomb-HP-Pavilion-TS-Sleekbook-14:~$ sudo tcpdump udp port 9
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on wlo1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
15:44:20.930362 IP box.36037 > 192.168.1.255.discard: UDP, length 102
15:44:20.932528 IP box.36037 > 192.168.1.255.discard: UDP, length 102
15:44:20.934762 IP box.36037 > 192.168.1.255.discard: UDP, length 102
15:44:20.936929 IP box.36037 > 192.168.1.255.discard: UDP, length 102
15:44:20.939084 IP box.36037 > 192.168.1.255.discard: UDP, length 102
15:44:20.943370 IP box.36037 > 192.168.1.255.discard: UDP, length 102
15:44:20.945549 IP box.36037 > 192.168.1.255.discard: UDP, length 102
15:44:20.947802 IP box.36037 > 192.168.1.255.discard: UDP, length 102
15:44:20.951843 IP box.36037 > 192.168.1.255.discard: UDP, length 102
15:44:20.954025 IP box.36037 > 192.168.1.255.discard: UDP, length 102
^C
10 packets captured
10 packets received by filter
0 packets dropped by kernel
````

We can see it is the box which sends the WOL packet via a broadcast IP `192.168.1.255`

 We also tested that if proxy WOL is disabled no packed is received inside the LAN.

### Use Wireshark 

We will do an [Android WOL](#android-wol).

And check wireshark.

![](./media/wireshark-screenshoft-wol-from-2022-08-07%2015-59-32.png)

We can see all the network stack.
And the both broadcast at udp level (`192.168.1.255`) and Ethernet level (`Ethernet II [...] Dst: Broadcast (ff:ff:ff:ff:ff:ff)`)

You can see corresponding export in [wireshark-export](./media/wireshark-export-android-wol-192-168-1-255.txt). (`File` > `Export packet dissection` > `As plain text`).

If we use [etherwake](#etherwake), we will have only ethernet frame as expected. This can be seen in this [export](./media/wireshark-export-etherwake.txt).

### QManager APP 

It can do WOL on LAN only, it is proposed when app can not manage to connect to NAS.

### In practise

- Use Android with [setup-2b-wow-with-cloud-server](#setup-2b-wow-with-cloud-server) from anywhere
- Use Laptop [etherwake](#etherwake)  inside LAN 

````shell
sudo etherwake -i wlo1  24:5e:be:3f:2b:f7 
sudo etherwake -i wlo1  192.168.1.88 # IF ethers file 
````
<!-- tested OK -->


<!-- worked via web ui, will not make test-->
<!-- WOL/WOW concluded OK STOP HERE -->
<!-- disclosed mac@ OK:https://security.stackexchange.com/questions/67893/is-it-dangerous-to-post-my-mac-address-publicly#:~:text=Disclosing%20the%20MAC%20address%20in,you%20and%20your%20immediate%20gateway). 
But anybody can turn on my NAS when WOL proxy is activated - set pwd not possible with my model - https://wiki.dd-wrt.com/wiki/index.php/WOL osef -->

We can use [VPN](README.md#configure-a-vpn-to-lan-via-nas) as alternative for WOW (not tested) via Android or not.

<!-- ok re-concluded STOP --> 