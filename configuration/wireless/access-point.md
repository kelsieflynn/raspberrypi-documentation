# Setting up a Raspberry Pi as an access point in a standalone network (NAT)

The Raspberry Pi can be used as a wireless access point, running a standalone network. This can be done using the inbuilt wireless features of the Raspberry Pi 3 or Raspberry Pi Zero W, or by using a suitable USB wireless dongle that supports access points. 

Note that this documentation was tested on a Raspberry Pi 3, and it is possible that some USB dongles may need slight changes to their settings. If you are having trouble with a USB wireless dongle, please check the forums.

To add a Raspberry Pi-based access point to an existing network, see [this section](#internet-sharing).

In order to work as an access point, the Raspberry Pi will need to have access point software installed, along with DHCP server software to provide connecting devices with a network address. Ensure that your Raspberry Pi is using an up-to-date version of Raspbian (dated 2017 or later).

Use the following to update your Raspbian installation:
```
sudo apt-get update
sudo apt-get upgrade
```
Install all the required software in one go with this command: 
```
sudo apt-get install dnsmasq hostapd
```
Since the configuration files are not ready yet, turn the new software off as follows: 
```
sudo systemctl stop dnsmasq
sudo systemctl stop hostapd
```

## Configuring a static IP

We are configuring a standalone network to act as a server, so the Raspberry Pi needs to have a static IP address assigned to the wireless port. This documentation assumes that we are using the standard 192.168.x.x IP addresses for our wireless network, so we will assign the server the IP address 192.168.4.1. It is also assumed that the wireless device being used is `wlan0`.

To configure the static IP address, edit the dhcpcd configuration file with: 
```
sudo nano /etc/dhcpcd.conf
```
Go to the end of the file and edit it so that it looks like the following:
```
interface wlan0
    static ip_address=192.168.4.1/24

```
Now restart the dhcpcd daemon and set up the new `wlan0` configuration:
```
sudo service dhcpcd restart

```

## Configuring the DHCP server (dnsmasq)

The DHCP service is provided by dnsmasq. By default, the configuration file contains a lot of information that is not needed, and it is easier to start from scratch. Rename this configuration file, and edit a new one:

```
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig  
sudo nano /etc/dnsmasq.conf
```

Type or copy the following information into the dnsmasq configuration file and save it:

```
interface=wlan0      # Use the require wireless interface - usually wlan0
  dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
```

So for `wlan0`, we are going to provide IP addresses between 192.168.4.2 and 192.168.4.20, with a lease time of 24 hours. If you are providing DHCP services for other network devices (e.g. eth0), you could add more sections with the appropriate interface header, with the range of addresses you intend to provide to that interface.

There are many more options for dnsmasq; see the [dnsmasq documentation](http://www.thekelleys.org.uk/dnsmasq/doc.html) for more details.

## Configuring the access point host software (hostapd)

You need to edit the hostapd configuration file, located at /etc/hostapd/hostapd.conf, to add the various parameters for your wireless network. After initial install, this will be a new/empty file.

```
sudo nano /etc/hostapd/hostapd.conf
```

Add the information below to the configuration file. This configuration assumes we are using channel 7, with a network name of NameOfNetwork, and a password AardvarkBadgerHedgehog. Note that the name and password should **not** have quotes around them. The passphrase should be between 8 and 64 characters in length.

```
interface=wlan0
driver=nl80211
ssid=NameOfNetwork
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=AardvarkBadgerHedgehog
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

We now need to tell the system where to find this configuration file.

```
sudo nano /etc/default/hostapd
```

Find the line with #DAEMON_CONF, and replace it with this:

```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

## Start it up

Now start up the remaining services:

```
sudo systemctl start hostapd
sudo systemctl start dnsmasq
```
### Add routing and masquerade

Edit /etc/sysctl.conf and uncomment this line:
```
net.ipv4.ip_forward=1
```

Add a masquerade for outbound traffic on eth0:
```
sudo iptables -t nat -A  POSTROUTING -o eth0 -j MASQUERADE
```
Save the iptables rule.
```
sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
```

Edit /etc/rc.local and add this just above "exit 0" to install these rules on boot.
```
iptables-restore < /etc/iptables.ipv4.nat
```
Reboot

Using a wireless device, search for networks. The network SSID you specified in the hostapd configuration should now be present, and it should be accessible with the specified password.

If SSH is enabled on the Raspberry Pi access point, it should be possible to connect to it from another Linux box (or a system with SSH connectivity present) as follows, assuming the `pi` account is present:

```
ssh pi@192.168.4.1
```

By this point, the Raspberry Pi is acting as an access point, and other devices can associate with it. Associated devices can access the Raspberry Pi access point via its IP address for operations such as `rsync`, `scp`, or `ssh`.

## ----------------------------------------------------------------------------------
<a name="internet-sharing"></a>
# Setting up a Raspberry Pi as an access point using bridge mode.(Unofficial)

One common use of the Raspberry Pi as an access point is to provide wireless connections to a wired Ethernet connection, so that anyone logged into the access point can access the internet, providing of course that the wired Ethernet on the Pi can connect to the internet via some sort of router.

To do this, a 'bridge' needs to put in place between the wireless device and the Ethernet device on the access point Raspberry Pi. This bridge will pass all traffic between the two interfaces. Install the following packages to enable the access point setup and bridging.

```
sudo apt-get install hostapd bridge-utils
```

Since the configuration files are not ready yet, turn the new software off as follows: 

```
sudo systemctl stop hostapd
```

Bridging creates a higher-level construct over the two ports being bridged. It is the bridge that is the network device, so we need to stop the `eth0` and `wlan0` ports being allocated IP addresses by the DHCP client on the Raspberry Pi.

```
sudo nano /etc/dhcpcd.conf
```
Add `denyinterfaces wlan0` and `denyinterfaces eth0` to the end of the file (but above any other added `interface` lines) and save the file.


At cmd line:
Add a new bridge, which in this case is called `br0`.

```
sudo brctl addbr br0
```

Connect the network ports. In this case, connect `eth0` to the bridge `br0`.

```
sudo brctl addif br0 eth0
```

*Do not attempt add wlan0 manually, this was the 'old way', in the 'new way' the wlan0 interface is added dynamically when hostapd starts



Now the interfaces file needs to be edited to adjust the various devices to work with bridging. `sudo nano /etc/network/interfaces` make the following edits.

For dhcp on br0. Add the bridging information at the end of the file.

```
# Bridge setup
auto br0
iface br0 inet manual
bridge_ports eth0 wlan0

```    

## Bridge static ip options
To set a static ip for br0, keep 'interfaces' as above and then you can use the method above for NAT "static ip" with /etc/dhcpcd.conf for br0
or
Add the static br0 information to the interfaces file directly as below.


```
# Bridge setup
# change ip schema from 192.168.1.0 to yours
auto br0
iface br0 inet static
        bridge_ports eth0 wlan0
                address 192.168.1.31
                broadcast 192.168.1.255
                netmask 255.255.255.0
                gateway 192.168.1.1
```    


## hostapd configuration options with wireless 'N' and 20/40Mhz
The access point setup is similar to  that shown in the previous section. Except it has options for "n" use with 20/40  MHZ. Follow the instructions above to set up the `hostapd.conf` file, but add `bridge=br0` below the `interface=wlan0` line, and remove or comment out the driver line. The passphrase must be between 8 and 64 characters long. 

```
interface=wlan0
bridge=br0
#driver=nl80211
ssid=NameOfNetwork
hw_mode=g
#to prevent overlap with n/40mhz, spread out channels for sta's, they will need the extra 4 slots(channels) to right of assigned channel when in 40mhz 
channel=6
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=AardvarkBadgerHedgehog
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
##for wireless n
wme_enabled=1
ieee80211n=1
#for 40mhz
ht_capab=[HT40+][SHORT-GI-40][DSSS_CCK-40]
#for congested networks still use N but 20mhz, leave off completely if errors on some cards
#ht_capab=[HT20][SHORT-GI-20][RX-STBC1]

```


## Daemon checks

Make sure hostapd is configured properly.

```
sudo nano /etc/default/hostapd
```

Find the line with #DAEMON_CONF, and replace it with this:

```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

## Enable hostapd

```
sudo systemctl enable hostapd

```

## Now reboot the Raspberry Pi.

There should now be a functioning bridge between the wireless LAN and the Ethernet connection on the Raspberry Pi, and any device associated with the Raspberry Pi access point will act as if it is connected to the access point's wired Ethernet.

The `ifconfig` command will show the bridge, which will have been allocated an IP address via the wired Ethernet's DHCP server. The `wlan0` and `eth0` no longer have IP addresses, as they are now controlled by the bridge. It is possible to use a static IP address for the bridge if required, but generally, if the Raspberry Pi access point is connected to a ADSL router, the DHCP address will be fine.




## Bridge/hostapd final Checks 
Check your bridge status and make sure it shows both eth0 and wlan0
````
brctl show 

````

If it does not , try restarting hostapd manually and look for process
````
systemctl restart hostapd
````

If it still doesnt start try checking the logs to get some info.
````
systemctl -l status hostapd
````

Here's a good working output example when using the builtin sysv startup script at /etc/init.d/hostapd

````
root@raspberrypi:/etc/hostapd# systemctl -l status hostapd
● hostapd.service - LSB: Advanced IEEE 802.11 management daemon
   Loaded: loaded (/etc/init.d/hostapd; generated; vendor preset: enabled)
   Active: active (running) since Sun 2018-06-17 00:27:29 EDT; 51s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 4729 ExecStop=/etc/init.d/hostapd stop (code=exited, status=0/SUCCESS)
  Process: 4736 ExecStart=/etc/init.d/hostapd start (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/hostapd.service
           └─4742 /usr/sbin/hostapd -B -P /run/hostapd.pid /etc/hostapd/hostapd.conf

Jun 17 00:27:27 raspberrypi systemd[1]: Starting LSB: Advanced IEEE 802.11 management daemon...
Jun 17 00:27:29 raspberrypi hostapd[4736]: Starting advanced IEEE 802.11 management: hostapd.
Jun 17 00:27:29 raspberrypi systemd[1]: Started LSB: Advanced IEEE 802.11 management daemon.

````

If you still can not get wlan0 on the bridge, just try to start it manually to prove your config and functionality.

````
hostapd /etc/hostapd/hostapd.conf

````

Then go to another terminal and check your bridge again.

````
brctl show

````


## sysv Daemon workarounds

*If it works this way but not automatically.
    Make sure /etc/init.d/hostapd is executable
    Force multiple restarts, sometimes until the first connection a multiple restart resolves a bug that prevents initial startup. *I discovered this after I gave up once and ditched the /etc/init.d/hostapd sysv and used a hostapd.service.
    It should not be needed if you keep trying. But, If you cant get yours to working bridging after the methods suggested, heres the hostapd.service to replace the sysv one.
 
Make sure the old sysv one is dead. Redundant but safe.

````
systemctl disable hostapd

````

````
update-rc.d hostapd remove

````

 
 Move /etc/init.d/hostapd out to new location, mark not executable and rename such as /root/hostapd.debian9u1.sysv.version

````
mv /etc/init.d/hostapd /root/hostapd.debian9u1.sysv.version
chmod -x /root/hostapd.debian9u1.sysv.version
````

## New systemd service for hostapd
Then install a new file in /etc/systemd/system/hostapd.service
  
  ````
[Unit]
Description=Hostapd IEEE 802.11 AP, IEEE 802.1X/WPA/WPA2/EAP/RADIUS Authenticator
After=network.target

[Service]
Type=forking
PIDFile=/run/hostapd.pid
ExecStart=/usr/sbin/hostapd /etc/hostapd/hostapd.conf -P /run/hostapd.pid -B

[Install]
WantedBy=multi-user.target
````

Do a systemd daemon-reload
````
systemctl daemon-reload
````



Enable the new systemd hostapd.service

````
systemctl enable hostapd
````

Start the new systemd hostapd.service
````
systemctl start hostapd.service #or just systemctl start hostapd
````

Here is an example of a working hostapd running with systemd

````
root@raspberrypi:/etc/systemd/system# systemctl status hostapd
● hostapd.service - Hostapd IEEE 802.11 AP, IEEE 802.1X/WPA/WPA2/EAP/RADIUS Authenticator
   Loaded: loaded (/etc/systemd/system/hostapd.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2018-06-17 00:44:35 EDT; 12s ago
  Process: 4881 ExecStart=/usr/sbin/hostapd /etc/hostapd/hostapd.conf -P /run/hostapd.pid -B (code=exited, status=0/SUCCESS)
 Main PID: 4882 (hostapd)
   CGroup: /system.slice/hostapd.service
           └─4882 /usr/sbin/hostapd /etc/hostapd/hostapd.conf -P /run/hostapd.pid -B

Jun 17 00:44:34 raspberrypi systemd[1]: Starting Hostapd IEEE 802.11 AP, IEEE 802.1X/WPA/WPA2/EAP/RADIUS Authenticator...
Jun 17 00:44:34 raspberrypi hostapd[4881]: Configuration file: /etc/hostapd/hostapd.conf
Jun 17 00:44:35 raspberrypi hostapd[4881]: wlan0: interface state UNINITIALIZED->HT_SCAN
Jun 17 00:44:35 raspberrypi systemd[1]: Started Hostapd IEEE 802.11 AP, IEEE 802.1X/WPA/WPA2/EAP/RADIUS Authenticator.

`````


## Check your bridge
````
brctl show 
````


Once hostapd is up and you see both interfaces in brctl show. Do a test connect with a client and
then check the following command on the pi WAP.

````
hostapd_cli all_sta

````

You should see information about your client.
If you get an error such as:
"Failed to connect to hostapd - wpa_ctrl_open:..."

Check/Add and or make sure you have the following at the top of /etc/hostapd/hostapd.conf

````
ctrl_interface=/var/run/hostapd
ctrl_interface_group=0
````




*Note in bridge mode you can have a external dhcpd server provide your addresses as needed or have it on the same server.
*Also note in bridge mode, the bridge is transparent to clients and clients can seach other across interfaces both wired and wireless, unless configured "not to", manually.
