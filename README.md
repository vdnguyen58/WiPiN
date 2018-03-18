# WiPiN
Broadcasting your VPN using Raspberry Pi as an Access Point

This tutorial will help you building a VPN-enabled wireless network. If you subscribe for a VPN service, they you'll find yourself comfortable with their provided VPN client such as VyprVPN, TorGuard etc. However, mostly VPN service provider supports OpenVPN protocol as well. Let's try it out with a Raspberry Pi to see how they're working together.

## Hardware requirements:
- 1 x Raspberry Pi. I prefer model 3 for built-in wifi capability. If you're using older model, please refer to [this link](https://elinux.org/RPi_USB_Wi-Fi_Adapters). Please note the wifi dongle must support AP mode https://wiki.gentoo.org/wiki/Hostapd

```shell
root #iw list | grep "Supported interface modes" -A 8
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

```shell
$sudo apt-get update
$sudo apt-get install hostapd isc-dhcp-server
```

### Configure DHCP server

Backup the current dhcpd config
```shell
$sudo cp /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.backup
```
Edit the config file
```shell
$sudo nano /etc/dhcp/dhcpd.conf
```
Tell this DHCP that it's the main DHCP server for the local network by uncomment this line:
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
Next, we need to tell which interface should be listening and responding to DHCP requests, aka managing the network. Edit the config file `isc-dhcp-server`:
```shell
$sudo nano /etc/default/isc-dhcp-server
```
Adding `wlan0` to `INTERFACES`
```
INTERFACES="wlan0"
```
In order to let `wlan0` to be the managing interface for the wireless network, it should be configured with static ip.

Backup the config file:
```
$sudo cp /etc/network/interfaces /etc/network/interfaces.backup
```
Edit the interface:
```
$sudo nano /etc/network/interfaces
```
Define wlan0 as static and eth0 as dhcp:
```
allow-hotplug eth0
iface eth0 inet dhcp

allow-hotplug wlan0
iface wlan0 inet static
  address 172.17.10.1
  netmask 255.255.255.0
  post-up iw dev $IFACE set power_save off
```
Close the file and verify the config by bringing the wlan0 down then up. You might need to try this with eth0 as well:
```
$sudo ifdown wlan0
$sudo ifup wlan0
```
... And check the information with `ifconfig` command.
