
# mikrotik-starlink-ipv6
A quick and easy IPv6 configuration for Mikrotik and Starlink

The overall goal of this is to set a couple variables, execute the script and have working IPv6 from Starlink on your MikroTik router and the rest of your network.

Key features:
- Routing is automatic, no need for a ::0/0 default route.
- Sane firewall rules, feel free to modify to fit your needs.
- No need for a IPv6 DHCP server. Network clients with IPv6 enabled should get globally routable addresses automatically.
- Passes https://ipv6-test.com/ test with full 20 out of 20 score

## Prerequisites
1. Starlink Internet
1. MikroTik router with RouterOS 7.7+
1. Blank IPv6 sections:
    - DHCP Client
    - DHCP Server
    - DHCP Relay
    - Firewall and Address List
    - Pool
    - No global addresses defined (link-local are ok)
1. Working winbox or ssh access.
    - For Mac users, you can use winbox via wine, [see section below](#winbox-on-mac).

## Configure IPv6
The following commands can be executing using winbox terminal or ssh.

### Step 1 - Set Variables
Two variables that must be set correctly for this to be successful. For me, I have Starlink plugged in to `ether1` on my router. Additionally, I have my LAN interface set to my main `bridge`. Compare these values to your Mikrotik configuration and update as needed.

```sh
:global StarlinkInterface "ether1";
:global LANInterface "bridge"
```

### Step 2 - Execute Script

The order here is important, *do not change it*. After activating the DHCP client, there is a 5 second delay to allow it time to bind. Later parts of the script are dependent upon values that only become available after the client is in a bound state.

```sh
/ipv6 settings
set accept-redirects=no accept-router-advertisements=yes disable-ipv6=no forward=yes max-neighbor-entries=8192

/ipv6 dhcp-client
add add-default-route=no dhcp-options="" dhcp-options="" disabled=no interface="$StarlinkInterface" pool-name=starlink-v6 pool-prefix-length=64 prefix-hint=::/0 rapid-commit=no request=prefix use-interface-duid=yes use-peer-dns=yes

:delay 5000ms

/ipv6 address
add address=::2/64 advertise=yes disabled=no eui-64=no from-pool=starlink-v6 interface="$LANInterface" no-dad=no

/ipv6 firewall address-list
add address=::/128 comment="defconf: unspecified address" disabled=no dynamic=no list=bad_ipv6
add address=::1/128 comment="defconf: lo" disabled=no dynamic=no list=bad_ipv6
add address=fec0::/10 comment="defconf: site-local" disabled=no dynamic=no list=bad_ipv6
add address=::ffff:0.0.0.0/96 comment="defconf: ipv4-mapped" disabled=no dynamic=no list=bad_ipv6
add address=::/96 comment="defconf: ipv4 compat" disabled=no dynamic=no list=bad_ipv6
add address=100::/64 comment="defconf: discard only " disabled=no dynamic=no list=bad_ipv6
add address=2001:db8::/32 comment="defconf: documentation" disabled=no dynamic=no list=bad_ipv6
add address=2001:10::/28 comment="defconf: ORCHID" disabled=no dynamic=no list=bad_ipv6
add address=fe80::/10 disabled=no dynamic=no list=prefix_delegation
add address=[/ipv6/dhcp-client get value-name=dhcp-server-v6 number=$StarlinkInterface] disabled=no dynamic=no list=prefix_delegation comment="dhcp6 client server value"

/ipv6 firewall filter
add action=accept chain=input dst-port=5678 protocol=udp
add action=accept chain=input comment="defconf: accept established,related,untracked" connection-state=established,related,untracked
add action=drop chain=input comment="defconf: drop invalid" connection-state=invalid
add action=accept chain=input comment="defconf: accept ICMPv6" protocol=icmpv6
add action=accept chain=input comment="defconf: accept UDP traceroute" port=33434-33534 protocol=udp
add action=accept chain=input comment="defconf: accept DHCPv6-Client prefix delegation." dst-port=546 protocol=udp src-address-list=prefix_delegation
add action=drop chain=input comment="defconf: drop everything else not coming from LAN" in-interface="!$LANInterface"
add action=accept chain=forward comment="defconf: accept established,related,untracked" connection-state=established,related,untracked
add action=drop chain=forward comment="defconf: drop invalid" connection-state=invalid
add action=drop chain=forward comment="defconf: drop packets with bad src ipv6" src-address-list=bad_ipv6
add action=drop chain=forward comment="defconf: drop packets with bad dst ipv6" dst-address-list=bad_ipv6
add action=drop chain=forward comment="defconf: rfc4890 drop hop-limit=1" hop-limit=equal:1 protocol=icmpv6
add action=accept chain=forward comment="defconf: accept ICMPv6" protocol=icmpv6
add action=accept chain=forward comment="defconf: accept HIP" protocol=139
add action=drop chain=forward comment="defconf: drop everything else not coming from LAN" in-interface="!$LANInterface"

/ipv6 nd
set [ find default=yes ] advertise-dns=no advertise-mac-address=yes disabled=no dns="" hop-limit=64 interface=all managed-address-configuration=yes mtu=1280 other-configuration=yes ra-delay=3s ra-interval=3m20s-8m20s ra-lifetime=30m ra-preference=medium

/ipv6 nd prefix default
set autonomous=yes preferred-lifetime=10m valid-lifetime=15m
```

## Winbox on Mac
Fortunately, the great winbox application can be run on a Mac quite easily with the help of [Homebrew](https://brew.sh/) and [Wine](https://www.winehq.org/).

1. ```brew install wine-stable```
2. Download latest [winbox](https://mt.lv/winbox64)
3. ```wine64 winbox.exe```

