# Pi-Router
**Turn your Raspberry Pi into a wireless router**

### Table of contents
1. [Disclaimer](#disclaimer)
2. [Requirements](#requirements)
3. [Setup](#setup)
4. [Configuring DHCP](#dhcp)
5. [Configuring hostapd](#hostapd)
6. [Configuring the firewall](#iptables)
7. [Finishing touches](#finishing)
8. [Troubleshooting](#troubleshooting)
9. [Sources](#sources)

<a name="disclaimer"></a>
### 1. Disclaimer
This is a collation of multiple tutorials and as such it's not a perfect solution and it is by no means guaranteed to work in every case for everyone. However, this is a solution that works for me.
This solution was intended to be initially set up on a home network, and then migrated to an industrial network with a captive portal. As such, a DHCP server will be used to route traffic within the new WiFi network, rather than using the "WAN" DHCP server.
I will try to help you if you encounter any issues but your best bet is to read through the sources at the bottom of this file.

<a name="requirements"></a>
### 2. Requirements
- Raspberry Pi 3 Model B
- A fresh installation of Raspbian Lite
- Internet access via Ethernet
- Optionally: A decent USB WiFi adapter (I used the TL-WN821N for this tutorial as I had it lying around)

<a name="setup"></a>
### 3. Setup
First ensure that the Pi has been configured and is all up to date (however, do not configure WiFi)
```sh
$ sudo raspi-config
$ sudo apt-get update
$ sudo apt-get upgrade
```
Next, install the necessary packages:
```sh
$ sudo apt-get install hostapd isc-dhcp-server
```
If any errors regarding the DHCP service appear, it is safe to ignore them for now

<a name="dhcp"></a>
### 4. Configuring DHCP
Firstly, backup the config file just in case you need to restore it later, and then open the file
```sh
$ sudo cp /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.default
$ sudo nano /etc/dhcp/dhcpd.conf
```
Once it opens comment out
```sh
option domain-name "example.org";
option domain-name-servers ns1.example.org, ns2.example.org;
```
and then uncomment
```sh
authoritative;
```
finally adding to the bottom of the file
```sh
subnet 192.168.42.0 netmask 255.255.255.0 {
  range 192.168.42.10 192.168.42.110;
  option broadcast-address 192.168.42.255;
  option routers 192.168.42.1;
  default-lease-time 600;
  max-lease-time 7200;
  option domain-name "local";
  option domain-name-servers 8.8.8.8, 8.8.4.4;
}
```
Note, the IP addresses can be changed to whatever you desire.

Now you need to find the name of the interface the dhcp server will be running on. This can be discovered by running
```sh
$ ifconfig -a
```
For me, this will be `wlan1`, so for the rest of the tutorial just adapt the solution to work for your adapter.
The DHCP server must be configured to use the adapter, so type
```sh
$ sudo nano /etc/default/isc-dhcp-server
```
and change the line with `INTERFACESv4=""` to read
```sh
INTERFACESv4="wlan1"
```

<a name="hostapd"></a>
### 5. Configuring hostapd
Edit the file by typing
```sh
$ sudo nano /etc/hostapd/hostapd.conf
```
and write inside it
```sh
interface=wlan1
ssid=WIFI_NETWORK_NAME
hw_mode=g
country_code=NZ
channel=11
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=WIFI_PASSWORD
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```
replacing the values of `ssid`, `country_code`, and `wpa_passphrase` with the desired values.
Now edit
```sh
$ sudo vim /etc/default/hostapd
```
and replace `#DAEMON_CONF=""` with
```sh
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```
ensuring that the line is uncommented.

<a name="iptables"></a>
### 6. Configuring the firewall
Start off by editing sysctl.conf
```sh
$ sudo nano /etc/sysctl.conf
```
and uncomment the line
```sh
net.ipv4.ip_forward=1
```
and then afterwards run
```sh
$ sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
```
so that the changes will come into effect immediately.
Now with the following commands we will set up iptables:
```sh
$ sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
$ sudo iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
$ sudo iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT
$ sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```
The iptables rules need to be updated after every boot so type
```sh
$ sudo nano /etc/rc.local
```
and on the line above `exit 0` write
```sh
iptables-restore < /etc/iptables.ipv4.nat
```

<a name="finishing"></a>
### 7. Final touches
Make sure all of the services are ready and enabled by typing
```sh
$ sudo update-rc.d hostapd enable
$ sudo update-rc.d isc-dhcp-server enable
```
Now all that's left to do is reboot!
```sh
$ sudo reboot
```

<a name="troubleshooting"></a>
### 8. Troubleshooting

#### **I can see the WiFi network but can't connect! (aka My DHCP server won't start at boot)**
This is an issue I ran into when making this tutorial. I made a bodge to make it work.
Start by making a new script
```sh
$ sudo mkdir /root/scripts
$ sudo nano /root/scripts/nwfix.sh
```
and write
```sh
#!/bin/bash

systemctl restart isc-dhcp-server.service
```
After saving and exiting, change the permissions so that it's runnable
```sh
$ sudo chmod 700 /root/scripts/nwfix.sh
```
Now we need a service to run this on boot. Let's make a new one
```sh
$ sudo nano /etc/systemd/system/nwfix.service
```
and type
```sh
[Unit]
Description=Fixes network init issues
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/root
ExecStart=/bin/bash /root/scripts/nwfix.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
Enable it and reboot
```sh
$ sudo systemctl start nwfix.service
$ sudo reboot
```
Once you're back in, check to see if it worked with
```sh
$ systemctl status nwfix.service
$ systemctl status isc-dhcp-server.service
```
The first one should be inactive (dead) and the other one active (running)

<a name="sources"></a>
### 9. Sources
[RouterBerry-Pi](https://github.com/smoscar/RouterBerry-Pi/)

[Instructables](https://www.instructables.com/id/Use-Raspberry-Pi-3-As-Router/)

[Turing a Pi into A Router](https://jacobsalmela.com/2014/05/19/raspberry-pi-and-routing-turning-a-pi-into-a-router/)

[How to use your Raspberry Pi as a wireless access point](https://thepi.io/how-to-use-your-raspberry-pi-as-a-wireless-access-point/)

[Configuring Raspberry Pi as a Router](https://blog.erratasec.com/2016/10/configuring-raspberry-pi-as-router.html)

[Configuring dhcpcd in Raspbian Stretch](https://lb.raspberrypi.org/forums/viewtopic.php?t=191453)
