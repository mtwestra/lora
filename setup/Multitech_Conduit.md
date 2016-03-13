## Set up Multitech mLinux Conduit MTCDT-210L-US-EU-GB for The Things Network

This document describes how to set up a Multitech mLinux for The Things Network, using a mac. It is based on http://www.multitech.net/developer/software/mlinux/getting-started-with-conduit-mlinux/

This page assumes you have got:
* MTCDT-210L-US-EU-GB Multitech mLinux Conduit (non-cellular)
* MTAC-LORA-868 accessory card which goes into the Conduit.

1. Install the MTAC-LORA-868 card into the Conduit - the instructions are packed with the MTAC-LORA-868 card, or here: http://www.multitech.net/developer/products/accessory-cards/installing-an-accessory-card/

### Setting up the conduit
The instructions for setting up the Conduit are described here: http://www.multitech.net/developer/software/mlinux/getting-started-with-conduit-mlinux/

1. Connect your pc to the conduit by ethernet cable. On the PC, configure the network interface that is connected to the Conduit to be a static IP address within the range `192.168.2.2 - 192.168.2.254`. On a mac, this is done by going to the network settings, select the thunderbolt ethernet, and under 'configure IPv4', select 'manually'. Assign it an IP in the right range, for example `192.168.2.2`.

2. Open an SSH connection to `192.168.2.1`, which is the default static IP address of the conduit, with DHCP disabled. Credentials: username `root`, password `root`
```
ssh root@192.168.2.1
```

when prompted, enter the default password.

3. If a connection is established, the conduit’s terminal prompt appears:
```
root@mtcdt:~#
```

### Setting the Time Zone, Date, and Time
To set the timezone, date, and time:
1. Create a symbolic link from the zoneinfo file for your location to `/etc/localtime`:
```
ln -fs /usr/share/zoneinfo/Europe/Amsterdam /etc/localtime
```

You can find valid values for the zoneinfo here: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones

2. Update the date and time to the current time: `date "2016-03-13 12:25:45"`
3. Update the hardware clock: `hwclock -u -w`

### Set custom static IP Address
1. Network configuration is defined in `/etc/network/interfaces`. Some more info can be found here: https://wiki.debian.org/NetworkConfiguration, although I am not sure how much of this applies for this device. Use your favourite editor to open this file, for example `vi`:
```
vi /etc/network/interfaces
```

2. Update the `address` and `netmask` values in the section 'Wired interface' to reflect the static IP address you want to use. In my case, I'll use a static IP address `192.168.1.5`. This took some experimentation: for some reason my router didn't like the default IP address of 192.168.2.1.
```
# Wired interface
auto eth0
iface eth0 inet static
address 192.168.1.5
netmask 255.255.255.0
```

3. After saving the file, either reboot the device or issue: `ifdown eth0 && ifup eth0` (note: you’ll lose your ssh session by doing this).

4. Connect the conduit to your router by ethernet cable

5. If everything worked, you can now reach the conduit by issueing:
```
ping 192.168.1.5
```

where the ip address is the static IP address you set up above. The response of the ping command should be something like this:
```
PING 192.168.1.5 (192.168.1.5) 56(84) bytes of data.
64 bytes from 192.168.1.5: icmp_seq=1 ttl=64 time=0.387 ms
64 bytes from 192.168.1.5: icmp_seq=2 ttl=64 time=0.283 ms
```

6. You should now be able to establish an ssh connection again by issueing:
```
ssh root@192.168.1.5.
```

7. You can also check your internet settings by running `ifconfig`.

### Alternative: use DHCP
You can let your router issue an IP address to the Conduit by using DHCP.
1. On your router, make sure DHCP is enabled, and note down the range of addresses it assigns.

2. Configure DHCP by editing `/etc/udhcpd.conf`:
```
start      192.168.1.100
end        192.168.1.200
interface  eth0
option subnet   255.255.255.0
option router   192.168.1.1
option dns  8.8.8.8 # google's DNS server
```

Here, `start` and `end` describe the range of DHCP addresses as issued by the router. `router` is the internal IP address of the router.

3. Configure `/etc/network/interfaces` to use DHCP. Edit the lines below 'Wired interface' to read:
```
# Wired interface
auto eth0
iface eth0 inet dhcp
```

4. Start the dhcp deamon by running `mlinux-dhcpd restart`

5. One option is to set a static DHCP IP address in your browser, which is coupled to the MAC address of the conduit device. In this way, even while using DHCP, the same IP address will be issued always.

### Configure internet access via the ethernet port
1. To configure internet access via the ethernet port, add a `gateway` parameter to the `/etc/network/interfaces`, which becomes:
```
# Wired interface
auto eth0
iface eth0 inet static
address 192.168.1.5
netmask 255.255.255.0
gateway 192.168.1.1
```

Where the gateway is the IP of the network router.

2. To apply changes, either reboot the device or issue: `ifdown eth0 && ifup eth0` (note: you’ll lose your ssh session by doing this).

3. To test you have internet access from within the conduit, issue `ping 8.8.8.8`, which pings the google DNS server.

### Upgrading firmware
The proces how to upgrade the firmware of the Conduit is described in detail here: http://www.multitech.net/developer/software/mlinux/using-mlinux/flashing-mlinux-firmware-for-conduit/

We used the recommended 'Using Auto-Flash During Reboot' method.

### Upgrade LoRa server and Package forwarder
Next step is to upgrade the LoRa server and default package forwarder. The process is described here:
http://www.multitech.net/developer/software/mlinux/using-mlinux/upgrade-lora-server/

1. Download latest versions of Lora Network Server (0.0.9 at time of writing) and Lora Packet Forwarder (1.4.1-r8 at time of writing) from the download page: http://www.multitech.net/developer/downloads/

2. On the conduit, create a tmp directory:
```
mkdir tmp
```

3. Copy both packages from the PC to the Conduit using SCP:
```
scp lora-network-server_0.0.9.2-r0.0_arm926ejste_mlinux.ipk root@192.168.1.5:/home/root/tmp/lora-network-server_0.0.9.2-r0.0_arm926ejste_mlinux.ipk
scp lora-packet-forwarder_1.4.1-r8.16_arm926ejste.ipk root@192.168.1.5:/home/root/tmp/lora-packet-forwarder_1.4.1-r8.16_arm926ejste.ipk
```

4. On the conduit, go to the tmp directory, and install both packages using the `opkg` package manager (be sure to use the correct file names):
```
opkg install lora-packet-forwarder_1.4.1-r8.16_arm926ejste.ipk
opkg install lora-network-server_0.0.9.2-r0.0_arm926ejste_mlinux.ipk
```

5. On the conduit, create the folder to hold the configuration file:
```
mkdir /var/config/lora
```

6. Copy lora-network-server.conf from the provided sample file:
```
cp /opt/lora/lora-network-server.conf.sample /var/config/lora/lora-network-server.conf
```

7. Update the `/var/config/lora/lora-network-server.conf` as needed. This is described here: http://www.multitech.net/developer/software/lora/getting-started-with-lora-conduit-mlinux/. Most importantly, make sure the `frequencyBand` is set to `868` if you're in the EU. Secondly, provide a `name` and `passphrase`.

8. Start the lora server with `/etc/init.d/lora-network-server restart`

9. Start a mosquitto client: 'mosquitto_sub -t lora/+/+ -v'. This basically starts a mosquitto client which subscribes to the internal mosquitto broker, and shows any messages received.

At this point, you have a private LoRa gateway. You could continue here by following the blog of http://www.disgruntledrats.com/?p=929 to talk to an mDot, or the part 'Setting Up the mDot' described here: http://www.multitech.net/developer/software/lora/getting-started-with-lora-conduit-mlinux/. You can also play around with the examples described here: https://developer.mbed.org/platforms/MTS-mDot-F411/.

### Connecting to The Things Network: Convert the conduit to a package forwarder
In order to connect to The Things Network, our gateway needs to forward the incoming packages to the Things Network servers.

1. Restart the network server. This will generate a configuration file for the packet forwarder.
```
/etc/init.d/lora-network-server restart
```

2. Copy the generated packet forwarder configuration to the configuration folder.
```
cp /var/run/lora/1/global_conf.json /var/config/lora/global_conf.json
```

Note: please check carefully the path of the `global_conf.json` file in the `/var/run/lora/1` folder. In other descriptions, the '/1' part is missing.

3. As described in step 5 of http://thethingsnetwork.org/wiki/Installing-your-Multitech-mLinux-Conduit, download @kersing's poly packet forwarder from here: https://github.com/kersing/packet_forwarder/blob/master/multitech-bin/poly-packet-forwarder_2.1-r2_arm926ejste.ipk

4. Move the package file to the conduit by SCP
```
scp poly-packet-forwarder_2.1-r2_arm926ejste.ipk root@192.168.1.5:/home/root/tmp/poly-packet-forwarder_2.1-r2_arm926ejste.ipk
```

5. From within the `/tmp` folder, install the package:
```
opkg install poly-packet-forwarder_2.1-r2_arm926ejste.ipk
```

6. If you get the error message `* resolve_conffiles: Existing conffile /var/config/lora/global_conf.json is different from the conffile in the new package. The new conffile will be placed at /var/config/lora/global_conf.json-opkg.`. Make the new config file the active one by running
```
mv /var/config/lora/global_conf.json /var/config/lora/global_conf.json.bak
mv /var/config/lora/global_conf.json-opkg /var/config/lora/global_conf.json
```

7. Create file `/var/config/lora/local_conf.json`, and put this content in it:
```
  {
  / Put there parameters that are different for each gateway (eg. pointing one gateway to a test server while the others stay in production) /
  / Settings defined in global_conf will be overwritten by those in local_conf /
      "gateway_conf": {
      / you must pick a unique 64b number for each gateway (represented by an hex string) /
          "gateway_ID": "AA555A000004BABA",
          / Email of gateway operator, max 40 chars/
          "contact_email": "operator@gateway.tst",
          / Public description of this device, max 64 chars /
          "description": "Update me",
      / Enter VALID GPS coordinates below before enabling fake GPS /
          "fake_gps": false,
          "ref_latitude": 10,
          "ref_longitude": 20,
          "ref_altitude": -1
      }
  }
```

You need to change the following:

  * gateway_ID can be obtained by running `mts-io-sysfs show lora/eui`
  * ref_latitude, ref_logitude and ref_altitude can all be got from Google, or from a location app on your phone.
  * fake_gps set to true
  * contact_email
  * description

8. Edit the file `/etc/init.d/lora-network-server` and make the following changes:

* around line 17, change `basic_pkt_fwd` to `poly_pkt_fwd`
```
pkt_fwd=/opt/lora/poly_pkt_fwd
```

* around line 57, comment out the part where the network is started:
```
# start network server                                                   
#start-stop-daemon --start --background --make-pidfile \                 
#    --pidfile $net_server_pidfile --exec $net_server -- \               
#    -c $conf_file --lora-eui $lora_eui --lora-path $run_dir --db $conf_db \
#    --noconsole -l $net_server_log                                        
#sleep 2                                                                   
```

* around line 65, make sure the `-c` option is set to `-c $conf_dir`, instead of `-c $run_dir/1`
```
start-stop-daemon --start --background --make-pidfile \                    
--pidfile $pkt_fwd_pidfile --exec $pkt_fwd -- \                        
-c $conf_dir -l $pkt_fwd_log  
```

* around line 71, comment out the line where the network server is stopped:
  ```
  do_stop() {                                                                   
      echo -n "Stopping $NAME: "                                                
      #start-stop-daemon --stop --quiet --oknodo --pidfile $net_server_pidfile --retry 15
      start-stop-daemon --stop --quiet --oknodo --pidfile $pkt_fwd_pidfile --retry 5    
      rm -f $net_server_pidfile $pkt_fwd_pidfile
      echo "OK"
  }     
```
9. Restart the lora-network server: `/etc/init.d/lora-network-server restart`

10. You can check the log of the package forwareder by `tail /var/log/lora-pkt-fwd-1.log` (note the `-1` in there).

11. Go to http://thethingsnetwork.org/api/v0/gateways/, and search for your gateway ID. If all is well, it should be there!
