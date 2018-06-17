
<a name="internet-sharing"></a>
# Setting up a Raspberry Pi as an access point using bridge mode. (Unofficial)

One common use of the Raspberry Pi as an access point is to provide wireless connections to a wired Ethernet connection, so that anyone logged into the access point can access the internet, providing of course that the wired Ethernet on the Pi can connect to the internet via some sort of router.

The Raspberry Pi can be used as a private wireless access point as well, running a standalone network. See here: [access-point-NAT.md] 
https://github.com/kelsieflynn/raspberrypi-documentation/blob/master/configuration/wireless/access-point-NAT.md
 *Note: A private access point can be made in bridging mode too, but requires more configuration not covered here.

## Using Bridging mode
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

*Do not attempt add wlan0 manually, that was the 'old way', in the 'new way' the wlan0 interface is added dynamically when hostapd starts



Now the interfaces file needs to be edited to adjust the various devices to work with bridging. `sudo nano /etc/network/interfaces` make the following edits.

For dhcp on br0. Add the bridging information at the end of the file.

```
# Bridge setup
auto br0
iface br0 inet manual
bridge_ports eth0 wlan0

```    

## Bridge static ip options
To set a static ip for br0, keep 'interfaces' as above and add a stanza "static ip" with /etc/dhcpcd.conf for br0(see NAT version)
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
The access point setup for bridging is similar to NAT. Except in this version I have included options for "n" use with 20/40  MHZ. Follow the instructions above to set up the `hostapd.conf` file, but add `bridge=br0` below the `interface=wlan0` line, and remove or comment out the driver line. The passphrase must be between 8 and 64 characters long. 

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
