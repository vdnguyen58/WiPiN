# WiPiN
Broadcasting your VPN using Raspberry Pi as an Access Point

This tutorial will help you building a VPN-enabled wireless network. If you subscribe for a VPN service, they you'll find yourself comfortable with their provided VPN client such as VyprVPN, TorGuard etc. However, most VPN service provider supports OpenVPN protocol as well. Let's try it out with a Raspberry Pi to see how they're working together.

## Hardware requirements:
- 1 x Raspberry Pi. I prefer model 3 for built-in wifi capability. If you're using older model, please refer to [this link](https://elinux.org/RPi_USB_Wi-Fi_Adapters). Please note the wifi dongle must support AP mode https://wiki.gentoo.org/wiki/Hostapd

```shell
$sudo iw list | grep "Supported interface modes" -A 8
        Supported interface modes:
                 * IBSS
                 * managed
                 * AP
                 * AP/VLAN
                 * WDS
                 * monitor
                 * P2P-client
                 * P2P-GO
```

## Software requirements:
- VPN subscription. In this tutorial I'll use TorGuard as an example.
- hostapd: to host your wireless network.
- isc-dhcp-server: dhcp server.

## Setup

```
$sudo apt-get update
$sudo apt-get install hostapd isc-dhcp-server openvpn
```

### Configure DHCP server

Backup the current dhcpd config:
```
$sudo cp /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.backup
```
Edit `/etc/dhcp/dhcpd.conf` file, tell this DHCP that it's the main DHCP server to the local network by uncomment this line:
```
#authoritative;
```
Then add these configurations to the end of the file:
```
subnet 172.17.10.0 netmask 255.255.255.0 {
    range 172.17.10.10 172.17.10.50;
    option broadcast-address 172.17.10.255;
    option routers 172.17.10.1;
    default-lease-time 600;
    max-lease-time 7200;
    option domain-name "local";
    option domain-name-servers 8.8.8.8, 8.8.4.4;
}
```
This is saying we're assigning the subnet `172.17.10.10-172.17.10.50` and the default gateway for that network is `172.17.10.1`  
Next, we need to tell which interface should be listening and responding to DHCP requests, aka managing the network. Edit the config file `/etc/default/isc-dhcp-server`, add `wlan0` to `INTERFACESv4`
```
INTERFACESv4="wlan0"
```
In order to let `wlan0` to be the managing interface for the wireless network, it should be configured with static ip.  
Backup the interfaces:
```
$sudo cp /etc/network/interfaces /etc/network/interfaces.backup
```
Edit `/etc/network/interfaces`, define wlan0 as static and eth0 as dhcp:
```
allow-hotplug eth0
iface eth0 inet dhcp

allow-hotplug wlan0
iface wlan0 inet static
  address 172.17.10.1
  netmask 255.255.255.0
  post-up iw dev $IFACE set power_save off
```
Verify the config by bringing the wlan0 down then up. You might need to try this with eth0 as well:
```
$sudo ifdown wlan0
$sudo ifup wlan0
```
And doublecheck the information with `ifconfig` command.
```
$sudo ifconfig wlan0
wlan0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.10.1  netmask 255.255.255.0  broadcast 172.17.10.255
        ether e8:4e:06:45:43:a6  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

### Configure Access Point
Create `/etc/hostapd/hostapd.conf` file, put those information to this new file to define your wireless network:
```
interface=wlan0
hw_mode=g
channel=11
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
ssid=WiPiN
wpa_passphrase=1234567890
```
We need to tell hostapd where to look at the config file by editing `/etc/default/hostapd` default file. Add `/etc/hostapd/hostapd.conf` to `DAEMON_CONF` like this, don't forget to uncomment the line:
```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```
Setup traffic forwarding to forward traffic from your wireless network (the WiPiN we're defining) to the Internet over your Ethernet (eth0). Backup `sysctl.conf` file:
```
$sudo cp /etc/sysctl.conf /etc/sysctl.conf.backup
```
Then edit it by uncommenting this line:
```
#net.ipv4.ip_forward=1
```
You may want to activate it without rebooting:
```
$sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
```
Next is iptables's rules to translate the outgoing packets, allow related incoming packets, also allow these outgoing packets. We achieve these by defining the rules accordingly:
```
$sudo iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE
$sudo iptables -A FORWARD -i tun0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
$sudo iptables -A FORWARD -i wlan0 -o tun0 -j ACCEPT
```
Why `tun0`? Basically we're gonna to establish a tunnel for VPN access later on this tutorial, its name is `tun0`. Let's save these rules:
```
$sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```

### Configure OpenVPN Client
We're downloading the necessary configuration files for our client:
```
$cd /etc/openvpn
$sudo wget https://torguard.net/downloads/OpenVPN-UDP.zip
$sudo unzip -j OpenVPN-UDP.zip
```
Then add a line to each of the configuration files above to tell where to look for the credentials by issuing this shell script:
```
$sudo sed -i '/auth-user-pass/cauth-user-pass user.txt' *.ovpn
```
Next, create the file mentioned above `user.txt` and put your TorGuard username & password in each line, for example:
```
JohnDoe@example.com
CatDog123
```
You don't want anybody other than the author to access it, so it needs a better permission:
```
$sudo chmod 700 user.txt
```

### Put everything together
Let's make our services start on boot:
```
$sudo update-rc.d hostapd enable
$sudo update-rc.d isc-dhcp-server enable
```
Backup our rc.local file:
```
$sudo cp /etc/rc.local /etc/rc.local.backup
```
Add openvpn daemon & iptables rules above line `exit 0` in `rc.local` file :
```
sudo openvpn --daemon --cd /etc/openvpn --config TorGuard.USA-LA.ovpn
sudo iptables-restore < /etc/iptables.ipv4.nat
```
Now try rebooting to verify & troubleshoot.
```
$sudo reboot
```

### Verify
Firstly, the interfaces:
```
$sudo ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.15  netmask 255.255.255.0  broadcast 192.168.0.255
...
tun0: flags=4305<UP,POINTOPOINT,RUNNING,NOARP,MULTICAST>  mtu 1500
        inet 10.35.0.10  netmask 255.255.255.255  destination 10.35.0.9
...
wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.10.1  netmask 255.255.255.0  broadcast 172.17.10.255
```
Your output might have slightly different tun0 & eth0 ip address.

Next, confirm hostapd, dhcpd, and openvpn are running
```
$sudo systemctl status hostapd
● hostapd.service - LSB: Advanced IEEE 802.11 management daemon
   Loaded: loaded (/etc/init.d/hostapd; generated; vendor preset: enabled)
   Active: active (running) since Sun 2018-03-18 12:24:08 PDT; 2min 2s ago
```
```
$sudo systemctl status isc-dhcp-server
● isc-dhcp-server.service - LSB: DHCP server
   Loaded: loaded (/etc/init.d/isc-dhcp-server; generated; vendor preset: enabled)
   Active: active (running) since Sun 2018-03-18 12:30:18 PDT; 1s ago
```
```
$sudo systemctl status openvpn
● openvpn.service - OpenVPN service
   Loaded: loaded (/lib/systemd/system/openvpn.service; enabled; vendor preset: enabled)
   Active: active (exited) since Sun 2018-03-18 12:24:07 PDT; 8min ago
```

Finally, try connecting to your WiPiN wifi with the password provided. Accessing www.iplocation.net to verify your location if it matches with your vpn setting.
