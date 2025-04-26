visible
-- TITLE --
Multi-instance Pi-hole setup
-- TAGS --
Pi-hole
keepalived
raspberry
linux
high-availability
-- TLDR --
Setting up multiple instances of Pi-hole with sync and high-availability
-- CONTENT --
I heard a lot about [Pi-hole](https://pi-hole.net/) over the last couple of years but never had the time nor felt the inclination to do something with it.
That is... until yesterday.


I watched the explanations by [TechnoTim](https://www.youtube.com/watch?v=OcSBggDyeJ4) and [Wundertech](https://www.youtube.com/watch?v=6sznCZ7ttbI&t=945s) a couple of times and went to work. I am not going to bore you with repeating what is already explained in the videos, but I do want to explain the changes I made.

## My setup
I am running 2 instances of Pi-hole, on a Proxmox Ubuntu LXC and a RaspberryPi 4, the setup was nice and easy,  `nebula-sync` and `keepalived` are running.
All is good, now I have some configs I want to port from my previous setup.

## Configuration requirements

- Split-DNS
- DNS for non-local domains
- Resolve .test to my dev machine
- Configure DHCP with static hosts (and failover)

### Split-DNS
I am running 3CX as my local PBX and it requires you to use Split-DNS.
This means that when you resolve the FQDN (name) of the PBX internally it should give you the local (private) IP, but externally it should resolve to the real IP.

Go to your Pi-hole dashboard, click on `Settings`, `System`, in the right page on the top switch from `Basic` to `Expert`, in the menu on the left the option `All Settings` appears, choose this. Select the config options for the DNS server and scroll down to `dns.hosts`.
In the `Values` box enter the local ip followed by a space and the FQDN, 1 per line.
```elixir
192.168.0.200 local.3cx.be
```

Do not forget to click `Save & Apply`.

### Non-local domains
I have several VPN connections to remote networks and it is nice to be able to use the local names on those networks from my own computer.
Imagine being connected to the network of customerX who has a local domain of `customerx.lan` with nameserver on `10.0.0.1`.
Pi-hole has no immediate setting for enabling this, but underneath it uses `dnsmasq` and it allows us to give extra configurations in the `Miscellaneous` section.
Click on `Miscellaneous` at the top of the page and scroll down to `misc.dnsmasq_lines` and enter as `Values`:
```elixir
server=/customerx.lan/10.0.0.1
```

### Resolve .test to my dev machine
I have set up my dev machine so when I am working on an app I can open it in my browser with a `.test` extension, eg _https://dashboard.gratwifi.test_.
On my dev machine they all resolve to `127.0.0.1`, but I also want to be able to test from other devices on my network.
In the same `Values` box as above, add:
```elixir
address=/.test/192.168.0.100
```
This requires my dev machine to have the fixed IP `192.168.0.100` for it to work.

There are numerous other options you can add in this box, check the [dnsmasq manual](https://thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html) for more information. Remember that entering an invalid line will make your instance fail. If that happens and your admin interface is no longer available, fix the error in `/etc/pihole/pihole.toml` and restart Pi-hole.

### DHCP with static hosts

This should have been quite easy... The initial setup is, but the sync threw me a curve.
Go to the DHCP option in the `Settings` menu, enter your Start, End range, Router ip and optionnally the subnetmask, configure your lease time, mine is `1d` for 1 day. Scroll down to `Static DHCP configuration` and enter your static hosts.
```elixir
00:11:22:33:44:55,192.168.0.100,dev
```
My dev machine will get it's fixed ip by DHCP.

As always remember to `Save & Apply`.

That was all to setup the DHCP, at least that's what I thought...
Until I checked the configuration on my secondary Pi-hole, only to disover the config wasn't there.
Hmm... Ah... the instructions I followed set `SYNC_CONFIG_DHCP` to `false`, changed it to `true`, same for `SYNC_GRAVITY_DHCP_LEASES`, restarted `nebula-sync` (docker compose).

Yes the config is now there, wait..., that DHCP server is now also enabled, that is not what I want.

Check the [Github page](https://github.com/lovelaze/nebula-sync) and found I need to add `SYNC_CONFIG_DHCP_EXCLUDE=active` to exclude it, restarted again, checked the config. YES!

### High-availability

The config I initially followed had me install `nebula-sync` and `keepalived`, the first to sync the configurations, the latter to ensure 1 instance is always available on the main ip address.
That works fine for DNS, but not for DHCP. DHCP is a bit like `Highlander`, there can be only one.
Not entirely true, but close enough for my network. I do not want to configure non-overlapping ranges, and whatnot.
I want the DHCP server on the active instance to be, well active.

This requires me to make some changes to `keepalived`, it allows me to run a script using `notify`, change my config to:
```elixir
global_defs {
  enable_script_security
}
vrrp_instance VRRP_IP4 {
  state BACKUP
  interface eth0
  virtual_router_id 198
  priority 10
  advert_int 1
  unicast_src_ip 192.168.0.247
  unicast_peer {
    192.168.0.245
  }

  virtual_ipaddress {
    192.168.0.248/24
  }

  notify /usr/local/sbin/pihole_dhcp
}
```
Before restarting `keepalived` it is required to create the `keepalived_script` user.
```elixir
groupadd -r keepalived_script
useradd -r -s /sbin/nologin -g keepalived_script -M keepalived_script
```
The script `/usr/local/sbin/pihole_dhcp` should be owned by that user.
```elixir
#! /bin/sh
SID=$(curl -sk -X POST "https://localhost/api/auth" --data '{"password":"PASSWORD"}' | jq -r ".session.sid")
if [ "$3" = "MASTER" ]
then
  new_state=$(curl -sk --request PATCH -d '{"config": {"dhcp": {"active": true}}}' "https://localhost/api/config?sid=$SID" | jq -r ".config.dhcp.active")
else
  new_state=$(curl -sk --request PATCH -d '{"config": {"dhcp": {"active": false}}}' "https://localhost/api/config?sid=$SID" | jq -r ".config.dhcp.active")
fi
curl -k -X DELETE "https://localhost/api/auth?sid=$SID"
logger -t pihole DHCP active=$new_state
```

It gets the `SID` then calls the Pi-hole api to update the DHCP server and finally logs off to free the session, and logs the new state.

Now when my primary goes down, the secondary takes over and when restored, disables the DHCP again.

Nice!
