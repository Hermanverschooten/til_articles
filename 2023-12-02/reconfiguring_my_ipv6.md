visible
-- TITLE --
Reconfiguring my IPv6 
-- TAGS --
network
ubuntu
IPv6
-- TLDR --
My IPv6 configuration was in dire need of reconfiguration... this is my now working setup.
-- CONTENT --
# Reconfiguring my IPv6 in Ubuntu 22.04 LTS

I have been running public IPv6 for a number of years now, but it has always been a hassle.
It would work for weeks on end and just as suddenly stop working for hours. Rebooting modems, firewalls, ... till my hair turned gray.
I use IPv6 almost all the time for my web apps, so when it goes down I can no longer access the servers.

This week it started again, and this time it was different. I would get an IPv6 address on my WAN-interface, I got my assigned range, but no default gateway.
Wait not entirely true, the default gateway would appear after some time.  Naturally I then decided to reboot my firewall server to see if it would still work.
What kind of masochists are we IT-ers? Anyway it didn't work, I had to leave and was convinced the default gateway would reappear as it dit before.
But it didn't and I'd had it. There has to be a better way to do this.

My current setup: network config via `netplan`, `wide-dhcpv6-client` for getting the IPv6 and `radvd` for announcing it on my network.
I have a fixed IPv6 range from my provider (Telenet), and assigned `::1` to my LAN-interface.
This would make `wide-dhcpv6-client` complain that it could not set the IP.
But in retrospect I think that removing the IPv6 address in netplan started the current avalanche.

This morning before sunrise when everyone was still asleep, I set to it.

`/etc/netplan/01-network.yaml`
```elixir
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0f1:
      match:
        macaddress: 00:15:17:XX:XX:XX
      addresses:
        - 192.168.1.1/24
        - "2a02:1807:XXXX:XXXX::1/64"
      nameservers:
        addresses:
          - 127.0.0.1
      set-name: lan
    enp1s0f0:
      match:
       macaddress: 00:15:17:XX:XX:XY
      dhcp4: true
      dhcp6: false
      accept-ra: false
      set-name: telenet
```
`/etc/sysctl.conf`
```elixir
net.ipv4.ip_forward = 1
net.ipv6.conf.default.forwarding = 1
net.ipv6.conf.all.forwarding = 1
net.ipv6.conf.all.accept_ra = 2
net.ipv6.conf.telenet.accept_ra = 2
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
```
This enables forwarding for both IPv4 and IPv6, and forces the Linux kernel to accept router anouncements (RA).

`/etc/wide-dhcpv6/dhcp6c.conf`
```elixir
profile default
{
  information-only;

  request domain-name-servers;
  request domain-name;

  script "/etc/wide-dhcpv6/dhcp6c-script";
};

interface telenet {
   send rapid-commit;
   send ia-pd 0;
   send ia-na 0;
};

id-assoc pd 0 {
  prefix-interface lan {
    sla-len 8;
    sla-id 0;
    ifid 1;
  };
};

id-assoc na 0 {
};
```
My provider (Telenet) requires me to supply them with the `DUID` so they can recognize it is me that's connecting and give me the correct `/56` range.
To make this work with wide, I had to download a script `wide_mkduid.pl` that would generate a file that I had to put into place, so it would get used.
Initially I had a firewall appliance and the `DUID` was based on it's MAC address.

`/etc/radvd.conf`
```elixir
interface lan {
  AdvManagedFlag off;      # no DHCPV6 server here.
  AdvOtherConfigFlag off;  # not even for options;
  AdvSendAdvert on;
  AdvDefaultPreference high;
  AdvLinkMTU 1280;
  prefix ::/64 
  {
    AdvOnLink on;
    AdvAutonomous on;
  };
  RDNSS 2a02:1807:XXXX:XXXX::1 { };
};
```

This was the most-of-the-time working configuration, I wrote about it in [Github Gist](https://gist.github.com/Hermanverschooten/40c701b7f52e256502c9fe78473912e4) a few years back.
Funny thing is that when I was googling this morning, I got either my own gist as answer or someone on Stack Overflow pointing to it.
There do not seem to be that many resources on the net.

## Changes

In my `01-network.yaml` I removed the IPv6 address from the telenet interface. After rebooting wide stopped complaining and added my `::1`, but no routing.
I stopped and disabled wide, modified the netplan to have `dhcp6: true` and `accept-ra: true` for the telenet interface.
After reboot I got a IPv6 IP on the telenet interface, I got a default route, but no range for the lan and wide no longer starts as it needs the same port as netplan is now using, no not netplan but __networkd__.

After some more googling on networkd this time I found I had to create 2 files for setting up the correct information.

`/etc/systemd/network/10-netplan-enp1s0f0.network.d/override.conf`
```elixir
[Match]
Name=telenet

[DHCPv6]
PrefixDelegationHint=::/56
IAID=0
```

and
`/etc/systemd/network/10-netplan-enp1s0f1.network.d/override.conf`
```elixir
[Match]
Name=lan

[Network]
IPv6PrefixDelegation=dhcpv6
IPv6DuplicateAddressDetection=1
LinkLocalAddressing=ipv6

[DHCPv6PrefixDelegation]
SubnetId=0
Token=::1
Assign=true
```
The name of the file needs to match the names in `/run/systemd/network/` or the config will not be picked up and should contain the real interface name, inside you match on the given name.

We are now getting a `::/56` for our LAN-interface, but it is not the correct one.

I spent the next couple of hours fighting the system trying to come up with a way to make networkd send out the correct `DUID` but I failed.
The file is called `/etc/systemd/network.conf` and in the end it only contains:
```elixir
[DHCP]
DUIDType=link-layer
```

Don't believe the Ubuntu documentation that says you have to use `[DHCPv6]` because it is ignored!
The `DUIDType` can be `link-layer`, `link-layer-time`, `uuid` or `vendor` (default), and you should be able to overwrite the `DUID` by setting `DUIDRawData`, but nope.
I used `tcpdump -n -vvv (udp port 546 or 547) or icmp6` to spy on the outgoing packets, looking at the send `client-ID`, most of the time it not even resembled what I put into `DUIDRawData`.
I ran wide again with tcpdump running to see what it sends, `(client-ID: hwaddr/time type 1 time 694016330 001517XXXXXY)`, I was never able to even approach this.
Giving up I decided it would be better to update the `DUID` at Telenet, but what do I need to fill in? The tcpdump shows `(client-ID hwaddr type 1 001517XXXXXY)`, but I need to fill in a hexadecimal address.
`networctl status telenet` to the rescue, according to information found it should display the `DUID` and it does `DHCP6 Client DUID: DUID-LL:0001001517XXXXXY0000`. `DUID-LL` should be replaced with `00:03` and then all others should be paired and separated by a colon.
`00:03:00:01:00:15:17:XX:XX:XY:00:00`, filled it in an telenet, `systemctl restart systemd-networkd`, and... nothing. Damn, damn, damn!!!
Why are there 5 zeroes at the end of the `DUID`, maybe try without them, so 10 pairs instead of 12. Restart network and __yes!__

I now have IPv6 again with correct subnet on my LAN, with routing and this even after I reboot.

Another morning well spent.

Now all that's left is to completely remove wide-dhcpv6-client.
