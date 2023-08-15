# What?

`torxy` - A transparent HTTP/HTTPS-proxy which redirect requests to some domains to the TOR local SOCKS5 server.

# Why?

The typical solution is based on DNS server and marking packets going through the router. When a browser goes
to domain `example.com` a DNS server resolves its IPs, append them into ipset/iptable or set/nftables and put
some mark `42` on this conntrack. Than this conntrack can be redirected to a VPN or some proxy server.

At the first look this solution looks stable and solid-rock. But there is a problem - a domain can be hosted
on a server which also contains some other sites. And they can be unavailable when accessing through a VPN or proxy.

Is there a better solution? We can intercept requests from a browser in order to inspect on which domain they go.
And after that make a decision what to do next. In case of HTTPS we need to get a
[Server Name Indication](https://ru.wikipedia.org/wiki/Server_Name_Indication) from a request and parse it.

# Router setup

1. Add IP `169.254.254.254/32` to the loopback interface:

   `ip addr add 169.254.254.254/32 dev lo`

   It is necessary because a packet can not be NATed to `127.0.0.0/8` network. Linux kernel drops such packets
   as [martian](https://en.wikipedia.org/wiki/Martian_packet). But instead they can be NATed to `169.254.0.0/16`.

1. Add a DNAT rule to the firewall:

   iptables:

   ```
   iptables \
       -A PREROUTING \
       -s $LAN \
       -p tcp -m multiport --dports http,https \
       -j DNAT --to-destination 169.254.254.254:3128
   ```

   nftables:

   ```
   nft add table ip nat
   nft add chain ip nat PREROUTING { type nat hook prerouting priority dstnat; }
   nft add rule ip nat PREROUTING ip saddr "$LAN" tcp dport { 80,443 } counter dnat to 169.254.254.254:3128
   ```

   `$LAN` - local network address, for instance `10.193.68.0/24`. `3128` is a port which `torxy` is listening on.

1. If you want to browse `.onion` sites you need to override their addresses to the router address.
   In case of using `dnsmasq` the configuration line looks like this:

   `address=/onion/$ROUTER`

   Where `$ROUTER` is the local address of your router, for instance `10.193.68.1`.

# Rules

Rules stored in `/etc/torxy.rules`.

- Empty lines and lines starting with `#` are ignored.
- Each line should contains the only one rule.
- First matching rule wins.
- Rule is case insensitive.
- URL can not be used in rule.

Examples:

```
# Whole zone .onion.
.onion

# The only one site.
homedepot.com
```

"Match" means "contains", i.e.:

Rule: `example.com`

Domain: `example.com` => Matches.

Domain: `example.com.net` => Matches.

Domain: `example.org` => Not matches.

# Requirements

- Python >= 3.7;
- [dpkt](https://github.com/kbandla/dpkt)
- [systemd](https://github.com/systemd/python-systemd) (optionally);

# Performance

`torxy` is not intended to be as fast as possible. But on my router with Celeron J3160 and 8 Gb of RAM it handles
100 Mbit/s easily.

# Usage

See `torxy --help` for details.
Rules can be reloaded on `SIGHUP`.

# License

GPL.
