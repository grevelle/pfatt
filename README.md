# About

This repository allows bypassing the AT&T U-Verse fiber gateway using pfSense. This method utilizes [netgraph](https://www.freebsd.org/cgi/man.cgi?netgraph) which is a graph-based kernel networking subsystem of FreeBSD. This low-level solution was required to account for the unique issues surrounding bridging 802.1X traffic and tagging a VLAN with an id of 0. I've tested and confirmed this setup works with AT&T U-Verse Internet on the BGW210-700 residential gateway. It probably works with others too.

The netgraph method will allow you to fully utilize your own router and fully bypass your residential gateway. It survives reboots, re-authentications, IPv6 and new DHCP leases.

# Bypass Procedure

## Prerequisites

* Local or console access to pfSense
* pfSense 2.5.1+ running on amd64 architecture
* Two physical network interfaces on your pfSense server
* The MAC address of your EAP-TLS Identity (from residential gateway used for certificates)
* Valid certificates to perform EAP-TLS authentication (see Extracting Certificates)

## Notes

If you have valid certs that have been extracted from an authorized residential gateway device, you can utilize the native wpa_supplicant client in pfSense to perform 802.1X EAP-TLS authentication.

Note that EAP-TLS authentication authorizes the device, not the subscriber. Meaning, any authorized device (e.g., BGW210) can be used to authorize the link. It does not have to match the residential gateway assigned to your account. For example, a BGW210 purchased of eBay can authorize the link. The subscriber's service is authorized separately (probably by the DHCP MAC and/or ONT serial number).

In supplicant mode, the residential gateway can be permanently disconnected. We will use netgraph to tag our traffic with VLAN0 and spoof the MAC address from the residential gateway.

## High Level Steps

Before continuing to the setup, it's important to understand how this method works. This will make configuration and troubleshooting much easier.

1. Netgraph creates an interface for us called ngeth0. This interface is connected to vlan0 which is configured to tag all traffic as VLAN0 before sending it on to the ONT interface.
2. wpa_supplicant binds to ngeth0 and initiates 802.1X EAP-TLS authentication
3. pfSense can then be configured to use ngeth0 as the WAN interface.
4. Spoof the MAC address of the residential gateway and request a DHCP lease on ngeth0. The packets get tagged as VLAN0 and exit to the ONT.
5. Now the DHCP handshake should complete and we should be on our way!

See the comments and commands bin/pfatt.sh for details about the netgraph setup.

## Detailed Steps

1. Edit the following configuration variables in `bin/pfatt.sh` as noted below. `$RG_ETHER_ADDR` should match the MAC address of your Residential Gateway. AT&T will only grant a DHCP lease to the MAC they assigned your device.
    ```shell
    ONT_IF='xx0' # NIC -> ONT / Outside
    RG_ETHER_ADDR='xx:xx:xx:xx:xx:xx' # MAC address of Residential Gateway
    CA_PEM='xxx.pem' # Replace xxx with file name of gateway certificate
    CLIENT_PEM='xxx.pem' # Replace xxx with file name of gateway certificate
    PRIVATE_PEM='xxx.pem' # Replace xxx with file name of gateway certificate
    ```

2. Copy `bin/pfatt.sh` to `/root/bin` (or any directory) and make executable:
    ```
    ssh root@pfsense mkdir /root/bin
    scp bin/pfatt.sh root@pfsense:/root/bin/
    ssh root@pfsense chmod +x /root/bin/pfatt.sh
    ```

3. Extracting Certificates:
    Certificates can be extracted by the exploitation of the residential gateway to get a root shell. Here are instructions to do so in windows by [iwleonards](https://github.com/iwleonards/extract-mfg).

    Be careful, you have sole responsibility for not bricking your residential gateway.

    In case they are needed, here are firmware archives:
    * [1.0.29 Firmware](https://mega.nz/file/35lBkbzC#MTrKdt57SEuz81Tn3MBKm-o_s1zv643MLmxyKILjsk8)
    * [Firmware archive](https://drive.google.com/file/d/1AcP3gbjpZOsnGTFApQOlalLzpjidUDj4/view?usp=drivesdk)

4. Upload your extracted certs (see Extracting Certificates) to /conf/pfatt/wpa. You should have three files in the wpa directory as such. You may also need to match the permissions. Do this with `sudo chmod -R 0600 /conf/pfatt/wpa`
      ```
      [2.5.1-RELEASE][root@pfsense.knox.lan]/conf/pfatt/wpa: ls -al
      total 19
      drwxr-xr-x  2 root  wheel     5 Jan 10 16:32 .
      drwxr-xr-x  4 root  wheel     5 Jan 10 16:33 ..
      -rw-------  1 root  wheel  5150 Jan 10 16:32 ca.pem
      -rw-------  1 root  wheel  1123 Jan 10 16:32 client.pem
      -rw-------  1 root  wheel   887 Jan 10 16:32 private.pem
      ```
    
5. To start pfatt.sh script at the beginning of the boot process pfSense team recommends you use a package called shellcmd. Use pfSense package installer to find and install it. Once you have shellcmd package installed you can find it in Services > Shellcmd. Now add a new command and fill it up accordingly (make sure to select earlyshellcmd from a dropdown):
    ```
    Command: /root/bin/pfatt.sh
    Shellcmd Type: earlyshellcmd
    ```
    It should look like this:
    ![Shellcmd Settings](img/Shellcmd.png)

    This can also be accomplished by manually editing your pfSense /conf/config.xml file. Add <earlyshellcmd>/root/bin/pfatt.sh</earlyshellcmd> above. This method is not recommended and is frowned upon by pfSense team.

6. Connect cables:
    - `$ONT_IF` to ONT (outside)
    - `LAN NIC` to local switch (as normal)

7. Prepare for console access.

8. Reboot.

9. pfSense will detect new interfaces on bootup. Follow the prompts on the console to configure `ngeth0` as your pfSense WAN. Your LAN interface should not normally change. However, if you moved or re-purposed your LAN interface for this setup, you'll need to re-apply any existing configuration (like your VLANs) to your new LAN interface. pfSense does not need to manage `$ONT_IF`. I would advise not enabling those interfaces in pfSense as it can cause problems with the netgraph.

10. In the webConfigurator, configure the  WAN interface (`ngeth0`) to DHCP using the MAC address of your Residential Gateway.

If everything is setup correctly, netgraph should be bridging EAP traffic between the ONT and RG, tagging the WAN traffic with VLAN0, and your WAN interface configured with an IPv4 address via DHCP.

# IPv6 Setup

Once your netgraph setup is in place and working, there aren't any netgraph changes required to the setup to get IPv6 working. These instructions can also be followed with a different bypass method other than the netgraph method. Big thanks to @pyrodex1980's [post](http://www.dslreports.com/forum/r32118263-) on DSLReports for sharing your notes.

This setup assumes you have a fairly recent version of pfSense. I'm using 2.5.1.

**DUID Setup**

1. Go to _System > Advanced > Networking_
1. Configure **DHCP6 DUID** to _DUID-EN_
1. Configure **DUID-EN** to _3561_
1. Configure your **IANA Private Enterprise Number**. This number is unique for each customer and (I believe) based off your Residential Gateway serial number. You can generate your DUID using [gen-duid.sh](https://github.com/MonkWho/pfatt/blob/master/bin/gen-duid.sh), which just takes a few inputs. Or, you can take a pcap of the Residential Gateway with some DHCPv6 traffic. Then fire up Wireshark and look for the value in _DHCPv6 > Client Identifier > Identifier_. Add the value as colon separated hex values `00:00:00`.
1. Save

**WAN Setup**

1. Go to _Interfaces > WAN_
1. Enable **IPv6 Configuration Type** as _DHCP6_
1. Scroll to _DCHP6 Client Configuration_
1. Enable **Request only an IPv6 prefix**
1. Enable **DHCPv6 Prefix Delegation size** as _60_
1. Enable _Send IPv6 prefix hint_
1. Enable _Do not wait for a RA_
1. Save

**LAN Setup**

1. Go to _Interfaces > LAN_
1. Change the **IPv6 Configuration Type** to _Track Interface_
1. Under Track IPv6 Interface, assign **IPv6 Interface** to your WAN interface.
1. Configure **IPv6 Prefix ID** to _1_. We start at _1_ and not _0_ because pfSense will use prefix/address ID _0_ for itself and it seems AT&T is flakey about assigning IPv6 prefixes when a request is made with a prefix ID that matches the prefix/address ID of the router.
1. Save

If you have additional LAN interfaces repeat these steps for each interface except be sure to provide an **IPv6 Prefix ID** that is not _0_ and is unique among the interfaces you've configured so far.

**DHCPv6 Server & RA**

1. Go to _Services > DHCPv6 Server & RA_
1. Enable DHCPv6 server on interface LAN
1. Configure a range of ::0001 to ::ffff:ffff:ffff:fffe
1. Leave **Prefix Delegation Range** _blank_.
1. Configure **Prefix Delegation Size** to _64_
1. Save
1. Go to the _Router Advertisements_ tab
1. Configure **Router mode** as _Stateless DHCP_
1. Save

If you have additional LAN interfaces repeat these steps for each interface.

That's it! Now your clients should be receiving public IPv6 addresses via DHCP6.

# Troubleshooting

## Logging

Output from `pfatt.sh` and `pfatt-5268AC.sh` can be found in `/var/log/pfatt.log`.

## tcpdump

Use tcpdump to watch the authentication, vlan and dhcp bypass process (see above). Run tcpdumps on the `$ONT_IF` interface:
```
tcpdump -ei $ONT_IF
```
You should see some more EAP packets from the `$ONT_IF` interface as it negotiates 802.1/X EAP authentication.

Once that completes, watch `$ONT_IF` and `ngeth0` for DHCP traffic.
```
tcpdump -ei $ONT_IF port 67 or port 68
tcpdump -ei ngeth0 port 67 or port 68
```

Verify you are seeing 802.1Q (tagged as vlan0) traffic on your `$ONT_IF ` interface and untagged traffic on `ngeth0`.

Verify the DHCP request is firing using the MAC address of your Residential Gateway.

If the VLAN0 traffic is being properly handled, next pfSense will need to request an IP. `ngeth0` needs to DHCP using the authorized MAC address. You should see an untagged DCHP request on `ngeth0` carry over to the `$ONT_IF` interface tagged as VLAN0. Then you should get a DHCP response and you're in business.

If you don't see traffic being bridged between `ngeth0` and `$ONT_IF`, then netgraph is not setup correctly.

## Promiscuous Mode

`pfatt.sh` will put `$ONT_IF` in promiscuous mode via `/sbin/ifconfig $ONT_IF promisc`. I think this is necessary for everyone but I'm not sure. Turn it off if it's causing issues.

## netgraph

The netgraph system provides a uniform and modular system for the implementation of kernel objects which perform various networking functions. If you're unfamiliar with netgraph, this [tutorial](http://www.netbsd.org/gallery/presentations/ast/2012_AsiaBSDCon/Tutorial_NETGRAPH.pdf) is a great introduction.

Your netgraph should look something like this:

![netgraph](img/netgraph.png)

Try these commands to inspect whether netgraph is configured properly.

1. Confirm kernel modules are loaded with `kldstat -v`. The following modules are required:
    - netgraph
    - ng_ether
    - ng_eiface
    - ng_one2many
    - ng_vlan
    - ng_etf

2. Issue `ngctl list` to list netgraph nodes. Inspect `pfatt.sh` to verify the netgraph output matches the configuration in the script. It should look similar to this:
```
$ ngctl list
There are 5 total nodes:
  Name: vlan0           Type: vlan            ID: 000000a3   Num hooks: 2
  Name: ngeth0          Type: eiface          ID: 000000a6   Num hooks: 1
  Name: <unnamed>       Type: socket          ID: 00000006   Num hooks: 0
  Name: ngctl28740      Type: socket          ID: 000000ca   Num hooks: 0
  Name: bce0            Type: ether           ID: 0000006e   Num hooks: 1
```
3. Inspect the various nodes and hooks.

## Reset netgraph

`pfatt.sh` expects a clean netgraph before it can be ran. To reset a broken netgraph state, try this:

```shell
/usr/sbin/ngctl shutdown $ONT_IF:
/usr/sbin/ngctl shutdown vlan0:
/usr/sbin/ngctl shutdown ngeth0:
```

## pfSense

In some circumstances, pfSense may alter your netgraph. This is especially true if pfSense manages either your `$ONT_IF`. If you make some interface changes and your connection breaks, check to see if your netgraph was changed.

# Credits

- [aus](https://github.com/aus) - 31m9ujhbsRRZs4S64njEkw8ksFSTTDcsRU - For the original work
- [iwleonards](https://github.com/iwleonards/extract-mfg) - Residential gateway certificate extraction
