# Pi Router

A full featured router built on an SBC.
In this incarnation, it is a Raspberry Pi 4.

If you are following this as a guide,
I recommend attaching a monitor and keyboard to your Pi, as we will be messing with the network.
If you are real careful, you can do all this over the network via ssh.
But, I will not be giving directions for that method.

| Feature         | Provider                                                  |
| ------          | ------                                                    |
| Gateway         | Linux Kernel IPv4 Forwarding                              |
| Routing/NAT     | [iptables](https://wiki.archlinux.org/index.php/Iptables) |
| Port Forwarding | [iptables](https://wiki.archlinux.org/index.php/Iptables) |
| DHCP            | [Pi Hole (via FTLDNS (dnsmasq))](https://pi-hole.net/)    |
| DNS             | [Pi Hole (via FTLDNS (dnsmasq))](https://pi-hole.net/)    |
| DNS over HTTPS  | [cloudflared](https://github.com/cloudflare/cloudflared)  |
| Wifi AP         | [hostapd](https://w1.fi/hostapd/)                         |

## Hardware

- [Raspberry Pi 4](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/)
    - I use the 2GB model, and there is plenty of RAM to spare.
    - We want to use the model 4 specifically, since it has actual Gigabit Ethernet and a real USB 3 bus that we can use for our second Ethernet adapter.
- USB Gigabit Ethernet Adapter
    - There are [plenty of options](https://www.jeffgeerling.com/blogs/jeff-geerling/getting-gigabit-networking).
- SD Card
    - I use a 32 GB UHS-I card.
        - Space is not an issue, 8 GB should be sufficient.
        - Speed is not much of an issue, so I usaully compromise on UHS-I.

## Background

Here are some resources on different Networking/Linux topics.
You do not have to deeply read all (or even one) of these resources to understand this project.
I am presenting them as tools to provide basics or fill in some gaps in knowledge.
I have not fully read all of these resources, but I have either read sections or skimmed over them.

- [TCP/IP networking reference guide](http://www.penguintutor.com/linux/basic-network-reference)
    - A short reference guide on IP-level networking.
    - If you are coming from a place of little-to-no knowledge, I strongly suggest reading this.
        - Aside from a full read, this guide is also a great reference.
- [Understand the basics of Linux routing](https://www.techrepublic.com/article/understand-the-basics-of-linux-routing/)
    - A gentle introduction to routing in Linux.
- [Linux Advanced Routing & Traffic Control HOWTO](https://tldp.org/HOWTO/Adv-Routing-HOWTO/index.html)
    - A in-depth guide to routing in Linux.
- [Guide to IP Layer Network Administration with Linux](http://linux-ip.net/html/index.html)
    - A thorough guide on Linux IP Networking.
- [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/)
    - A well-written guide to network programming.
    - Since we are not doing any network programming, most of this will not be useful.
        - However, many of the earlier sections provide great definitions and basic concepts.
- [Network Configuration Arch Linux Wiki Page](https://wiki.archlinux.org/index.php/Network_configuration)
    - The Arch Linux wiki is, in my opinion, the best Linux distribution wiki and a great general Linux reference.
    - If you are using a different distribution for this project or want to use some different tools than the ones I used in this guide, the Arch Linux wiki may provide useful.

## Raspbian

For the router, I use Raspbian Buster Lite (2020-02-13) since the Pi-hole installer is made to work with Raspbian.
How you download and install it is up to you.

I will leave setting up the system up to your personal preference.
In this guide I will be following some more idiomatic Raspbian patterns.
Here are some things to make sure to do:
- Create a new user.
    - Give this user sudo.
    - Set a new password.
- Remove the default user.
- Configure through `raspi-config`.
    - Enable ssh.
    - Set localisation.
    - Expand the filesystem.
    - Set hostname.
- Update package index.
    - `sudo apt-get update`
- Install base packages and whatever else you want.
    - `sudo apt-get install git vim`

See Also:
- https://www.raspberrypi.org/downloads/raspbian/
- https://www.raspberrypi.org/documentation/installation/installing-images/linux.md

## Interfaces & Bridge

We will need to set up the network devices we will use on the router.
The main Ethernet port (`eth0`) on the Pi will be used for the WAN,
while the USB Ethernet adapter (`eth1`) and wireless NIC (`wlan0`) will be used for LAN.
To list interfaces and IPs, use:
```sh
ip a
```

### IP Forwarding

The first thing with configuring the interfaces is to enable IP forwarding:
```sh
sudo bash -c 'echo 1 > /proc/sys/net/ipv4/ip_forward'
```
To make this change persistent between reboots, we will need to uncomment/add the following line to `/etc/sysctl.conf`:
```ini
net.ipv4.ip_forward = 1
```

### Configuring Interfaces

To configure network devices in Raspbian, you can put individual files in the `/etc/network/interfaces.d/` directory.
I prefer to use one file per interface.
We will first create the files for the configuration, and then enable them all at the same time.
The exact configurations are in the `interfaces` directory, but here are the high-level points:
- The native Ethernet port, `eth0` will be used as the WAN (connection to the outside network).
    - This interface should use DHCP so other routers can assign our network an IP.
- The USB Ethernet port, `eth1`, will be used as part of the LAN (connection to the inside world).
    - This interface will be part of a bridge, so should not use DHCP or get a static IP.
- The Wifi, `wlan0`, will be used as part of the LAN (connection to the inside world).
    - This interface will be part of a bridge, so should not use DHCP or get a static IP.
- We will create a bridge, `lan`, that will bridge all the LAN connection.
    - We will need to assign a static IP (`192.168.10.1`) to the bridge, since it will be used as the gateway.

See Also:
- https://unix.stackexchange.com/questions/128439/good-detailed-explanation-of-etc-network-interfaces-syntax

### Creating the Bridge

First, install `bridge-utils`:
```sh
sudo apt-get install bridge-utils
```

To create the bridge, we can use the following command:
```sh
sudo brctl addbr lan
```

Then we can add our devices to the bridge:
```sh
sudo brctl addif lan eth1
```

Finally, you can show the bridge:
```sh
sudo brctl show
```
You should see something like:
```
bridge name     bridge id               STP enabled     interfaces
lan             8000.70886b8ce1dc       no              eth1
```

One thing to note is that we cannot add `wlan0` to the bridge directly.
`wlan0` will be added automatically by `hostapd` (which we will configure later) when the network is broadcast.

### Restarting the Network

Restart the network and reload all the devices with:
```sh
sudo systemctl restart networking.service
```

## IP Tables

Now we will set up all the `iptables` rules we need.
There are a lot of good resources that discuss `iptables` out there, and you should get the general idea before going through this section.

See Also:
- https://wiki.archlinux.org/index.php/Iptables
- https://www.youtube.com/watch?v=vbhr4csDeI4
- https://www.crybit.com/what-is-iptables-in-linux/
- https://www.youtube.com/watch?v=ldB8kDEtTZA

### Rules

The full rules are available in the `rules.v4` file.
We will be looking at each section/table at a time.
Tables start with a `*` and end with a `COMMIT`.

#### Filter

The filter table is mainly responsible for dropping packets.
There are three default chains here:
- `INPUT` - For packets that are coming into this system (the router).
- `FORWARD` - For packets that are routed through this system.
- `OUTPUT` - For packets generated from this system (the router).

For the `INPUT` and `FORWARD` chains, we will default to dropping packets.
But for for `OUTPUT` chain, we will default to accepting the packets (since they are from us).
```
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
```

We will always accept loopback packets.
```
-A INPUT -i lo -j ACCEPT
```

We will accept all packets destined for the router coming from the LAN.
```
-A INPUT -i lan -j ACCEPT
```

We will also allow packets coming from the WAN on connections that we have previously established.
Since we already allow all outgoing packets,
we will establish a connection with those and then these packets will be the followup.
```
-A INPUT -i eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

We will forward any packets either part of an existing connection or coming from the LAN.
```
-A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i lan -j ACCEPT
```

#### NAT

The Network Address Translation (NAT) table is responsible for changing IP addresses on incoming and outgoing packets.
This is the real "routing" part of a router.
In addition to the `INPUT` and `OUTPUT` chains we previously discussed, the NAT table also has two more chains:
- `PREROUTING` - For all packets before they have been routed.
- `POSTROUTING` - For all packets after they have been routed.

We will default to accepting packets on all chains (dropping will be done at the filter table):
```
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
```

The only real thing we do in the NAT table is enable NAT.
This rule will send all packets coming from the LAN and destined for the WAN to the special `MASQUERADE` target.
The `MASQUERADE` target will make it look like the network is a single machine to the WAN
while incoming packets can still be routed to the proper machines.
```
-A POSTROUTING -s 192.168.10.0/24 -o eth0 -j MASQUERADE
```

See Also:
- http://www.billauer.co.il/ipmasq-html.html

### Load Rules on Boot

By default, the tables are flushed on boot.
So to make them persistent, we will use the `iptables-persistent` package.
```sh
sudo apt-get install iptables-persistent
```
This will automatically load the rules in `/etc/iptables/rules.v4` at boot.
(There is also a files for IPv6 rules, but we are not using those.)

You can manually load rules with:
```sh
sudo bash -c 'iptables-restore < /etc/iptables/rules.v4'
```

### Test

This would be a good time to test your setup.
Connect with a wired or wireless device, assign a static IP like `192.168.10.100`, and try using the network.
Make sure you can ping the router, `192.168.10.1` and the outside world.

## Pi Hole

Pi Hole will handle DNS (with ad blocking) and DHCP for us.
The ad blocking portion of Pi Hole works by keeping a blacklist of domains that serves ads and dropping them at the DNS level.
This means that the ads don't get a change to load or even generate network traffic outside of the LAN.
The blacklists and whitelists for these domains are handled per-install and not by Pi Hole directly.

See Also:
- https://github.com/pi-hole/pi-hole

### Setup

The suggested way to install Pi Hole is to go through their installer:
```sh
git clone https://github.com/pi-hole/pi-hole.git
cd pi-hole/automated\ install/
sudo bash basic-install.sh
```

An ncurses GUI will open up and walk you through the installation.
Some key notes (do whatever you want for the other options):
 - For the interface, choose the bridge, `lan`.
 - Just choose any DNS you like for now (we will change is later).
 - Choose "No" to using the current network settings as a static address.
    - Instead use `192.168.10.1/21` for the IP address and `192.168.10.1` for the gateway.
 - I recommend installing the web GUI.

After the installer finishes, you should also set a better password for the web GUI:
```sh
sudo pihole -a -p
```

#### Web GUI

The web GUI for Pi Hole is pretty good, and it include a lot of useful graphs and stats.
So even though I generally prefer CLI, I think the GUI is worth installing.

Load up the GUI on a device on the LAN by going to: 192.168.10.1/admin

### DHCP

Go to the `Settings` -> `DHCP` tab in GUI.
Here you can enable DHCP (click the checkbox),
set the IP range (I use `192.168.10.100` to `192.168.10.200`),
set the default gateway (`192.168.10.1`),
and set static DHCP leases.
(Make sure to save at the bottom.)

With DHCP configured, you can now make sure all your LAN devices use DHCP.

### Reconfigure

If you ever mess up and need to reconfigure/reinstall your Pi Hole setup, you can use:
```sh
sudo pihole -r
```

## DNS over HTTPS

By default, DNS traffic is unencrypted.
There are several ways to encrypt it, and one is DNS over HTTPS (DoH).
In our system, we want clients to be able to make normal DNS query to the router,
which will then send off a DoH query and respond to the original DNS query with the response.

### Cloudflared

For our DoH provider, we will use Cloudflare.
They have put out an open source client, `cloudflared`, that will work with their DNS servers (`1.1.1.1` and `1.0.0.1`).
I am not a fan of only using two DNS servers, or a tool that is maintained by the same company that provides the servers.
However there are not any better options as of now, and Cloudflare has not done anything shady wrt DoH.

Unfortunately there are not any packages for `cloudflared` in the standard repos,
so we will have to download, install, and load on startup ourselves.
I suggest following Pi Hole's guide:
https://docs.pi-hole.net/guides/dns-over-https/

You can test that `cloudflared` is properly installed and running by doing:
```sh
dig @127.0.0.1 -p 5053 google.com
```

After you see everything is working, make sure to enable `cloudflared` on startup:
```sh
sudo systemctl enable cloudflared
```
If your installation method did not install a service,
this repository provides a sample Systemd unit called `cloudflared.service`.

See Also:
- https://developers.cloudflare.com/1.1.1.1/dns-over-https/cloudflared-proxy/
- https://docs.pi-hole.net/guides/dns-over-https/

### Enabling DoH in Pi Hole

Now that `cloudflared` is working, you can tell Pi Hole to use it as a DNS server.
Go to the `Settings` -> `DNS` section of the Pi Hole web GUI.
Now uncheck everything under "Upstream DNS Servers".
Check the box for "Custom 1 (IPv4)" and enter `127.0.0.1#5053` in the text field.
Don't forget to click the "Save" button at the bottom.
This will have Pi Hole use our local `cloudflared` service (on the default 5053 port) for DNS.
(Remember that we enabled all loopback packets in iptables.)

After enabling DoH, you can use the following links to test if it is working:
- https://1.1.1.1/help
- https://www.cloudflare.com/ssl/encrypted-sni/

## Wifi

Since our Raspberry Pi has a wireless NIC, we can set it up as an access point so clients can connect to our LAN using wifi.
(You can also skip all this and get a dedicated AP if you need serious wifi.)
The key here will be using `hostapd`, which will manage our AP.
You can use and modify the `hostapd.conf` file provided in this repo.
Note that in the config is the bridge that we have all our LAN interfaces on.
When `hostapd` brings up the AP, it add `wlan0` to the bridge.

Install `hostapd` and set the AP configuration (don't forget to set the SSID and passphrase):
```sh
sudo apt-get install hostapd
cp hostapd.conf /etc/hostapd/hostapd.conf
```

You will also need to configure the `hostapd` service to use this new config by setting this line in `/etc/default/hostapd`:
```ini
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

Unmask, start, and enable the `hostapd` service:
```sh
sudo systemctl unmask hostapd
sudo systemctl start hostapd
sudo systemctl enable hostapd
```

See Also:
- https://wiki.archlinux.org/index.php/Software_access_point#Wi-Fi_link_layer
- https://wireless.wiki.kernel.org/en/users/documentation/hostapd
- https://gist.github.com/renaudcerrato/db053d96991aba152cc17d71e7e0f63c

## Managing Ports

Our default setting is to reject all packets from the WAN that are not already part of an established connection.
If necessary, we can open/forward some ports so that those connections can be established from the WAN.

See Also:
- https://www.systutorials.com/port-forwarding-using-iptables/

### Opening a Router Port

To open a port (ssh on 22 in this example) to connect directly to the router, we can use the following rule on the filter table:
```
-A INPUT -i eth0 -p tcp --dport 22 -j ACCEPT
```
Note that we matched both the protocol (`tcp`) and port (`22`).

### Forwarding a Port

To forward a port (ssh on 22 in this example) to some other machine on the LAN, we can use the following rules in the nat table:
```
-A PREROUTING -i eth0 -p tcp --dport 22 -j DNAT --to 192.168.10.172:22
```
and filter table:
```
-A FORWARD -i eth0 -p tcp -d 192.168.10.172 --dport 22 -j ACCEPT
```

The first rule rewrites the address of the packet's target to be our designated LAN device,
and the second rule accepts this packet heading towards our designated address and port.
Note that we have to know the IP of the LAN device we are forwarding to.

Also note that we don't have to use the same port the packet was sent in using.
We can do any port translation we want here.
For example maybe we want to allow external ssh connections on port 23,
but then translate them to a specific computer on port 22.
This way, all machines on our network can be ssh'd into without exposing them directly
(these rules should be in their respective tables):
```
-A PREROUTING -i eth0 -p tcp --dport 23 -j DNAT --to 192.168.10.172:22
-A FORWARD -i eth0 -p tcp -d 192.168.10.172 --dport 22 -j ACCEPT
```
