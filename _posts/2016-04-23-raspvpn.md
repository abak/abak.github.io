---
layout: post
title: Raspberry Pi as a VPN wifi hotspot
---

This post is a notebook of the setup I use with one of my raspberry pi to get it running a wi-fi hotspot that routes all of its traffic through a VPN.

---

## Basic Setup : 

This setup uses a Raspberry Pi 3 model b, mostly because it has onboard Wi-Fi. I also conducted tests on a Raspberry Pi 2 model B with a Buffalo Wi-Fi dongle that supports the [`nl80211` driver](https://wireless.wiki.kernel.org/en/developers/documentation/nl80211).

The OS is an up-to-date Raspbian Jessie, which means that the init system is [systemd](https://www.freedesktop.org/wiki/Software/systemd/).

## Software Dependencies

The following needs to be installed : 

* openvpn : to connect to the vpn server
* hostapd : to create the wifi hotspot
* udhcpd : to give IP adresses to the devices connected to the hotspot

```
sudo apt-get install openvpn hostapd udhcpd
```

The package `iptables-persistent` can also be installed to provide an easier way to persist iptables rules.


## Wi-Fi hotspot

To set up the rapsi as a Wi-Fi hotspot, we need to set up the ethernet and the wireless interfaces. Only the latter needs to be set up with a static IP (since it will be the dhcp server for that network), but for convenience, I set both as static.


    auto lo
    iface lo inet loopback
    
    auto eth0
    allow-hotplug eth0
    iface eth0 inet static
            address 192.168.11.42
            netmask 255.255.255.0
            gateway 192.168.11.1
            dns-nameservers 8.8.8.8 8.8.4.4

    auto wlan0
    iface wlan 0 inet static 
            addres 192.168.42.1
            netmask 255.255.255.0

    up iptables-restore < /etc/iptables.nat.vpn.secure


The last line imports the iptable rules at startup. Those rules are here to perform NAT between the internet and the internal network, and to make sure that all traffic is routed through the vpn.


## VPN Login and configurations

The login informations for my VPN provider are stored in `/etc/openvpn/openvpn.login`. Given that, the configuration files provided by the vendor are edited to replace the default authentication by a file based one. This is usually done by replacing `auth-user-pass` by `auth-user-pass /etc/openvpn/openvpn.login`.

## IPtable Rules

The following set of IP tables rules are saved in `/etc/iptables.nat.vpn.secure` : 

     *nat
     :PREROUTING ACCEPT [4:711]
     :INPUT ACCEPT [4:711]
     :OUTPUT ACCEPT [0:0]
     :POSTROUTING ACCEPT [0:0]
     -A POSTROUTING -o tun0 -j MASQUERADE
     -A POSTROUTING -o tun0 -j MASQUERADE
     COMMIT
      
     *filter
     :INPUT ACCEPT [65:4284]
     :FORWARD ACCEPT [0:0]
     :OUTPUT ACCEPT [66:7192]
     -A INPUT -i tun0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
     -A INPUT -i tun0 -j DROP
     -A FORWARD -i tun0 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
     -A FORWARD -i eth0 -o tun0 -j ACCEPT
     -A FORWARD -i tun0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
     -A FORWARD -i wlan0 -o tun0 -j ACCEPT
     COMMIT

They serve 2 purposes. The first part is here to ensure that NAT is performed betzeen the inside and the outside world. The second part ensures that all outgoing connection is using the VPN.

## Start Everything on Boot

Everything gets started by systemd as follows :

    sudo systemctl start hostapd
    sudo systemctl enable hostapd
    
    sudo systemctl start openvpn@configuration
    sudo systemctl enable openvpn@configuration
    
    sudo systemctl start udhcpd
    sudo systemctl enable udhcpd

## Putting it all Together

An install script that automate all this process is available on [my github page](https://github.com/abak/raspvpn).
