# Overview

In this section,we will route traffic through "wireguard" for container. Sometimes we need this way to protect traffic between container and remote server. when the docker access the externel network, it should looks like below:

```
container <--> wireguard tunnel <--> remote server <--> Internet
```

It will involve installing wireguard on the host machine. Then we create a tunnel between client and server. The example configuration files are in the `client` and `server` directories.

# Practice

## Server

Just creating interface `wg0` by wg-quick.

```bash
$ wg-quick up wg0
```

## Client

First of all, we should create a bridge network with docker on the client side.

```bash
$ docker network create test
$ docker network inspect test # it will show the bridge network details, pay attention to the "Subnet"
```

In my case, the subnet is `172.19.0.0/16`.

I’m going to be creating an interface called `wg0`. Our beloved `DNS` and `Address` configurations found in `wg0.conf` have to be commented out as they are `wg-quick` specific.

Then, creating interface manually.

```bash
$ ip link add dev wg0 type wireguard
$ wg setconf wg0 /etc/wireguard/wg0.conf
$ ip address add 10.0.0.2/24 dev wg0
$ ip link set up dev wg0
$ iptables -t nat -A POSTROUTING -s 172.19.0.0/16 ! -d 172.19.0.0/16 -o wg0 -j MASQUERADE
$ printf 'nameserver %s\n' '10.0.0.1' | resolvconf -a tun.wg0 -m 0 -x
$ sysctl -w net.ipv4.conf.all.rp_filter=2 # 0 is ok
```

At this point, if your VPN is hosted externally you can test that the wireguard interface is working by comparing these two outputs:

```bash
# make sure your curl do not have proxy settings
$ curl ip.sb
$ curl --interface wg0 ip.sb
```

Now to route traffic for `test` through our new `wg0` interface:

```bash
$ ip rule add from 172.19.0.0/16 table 10
$ ip route add default via 10.0.0.2 table 10
```

Create a docker container to test.

```bash
$ docker run --rm --net=test -it alpine /bin/ash
```

Try `mtr dns.google` command.

Before:

```bash
                             My traceroute  [v0.93]
ec1d9d99af4c (172.19.0.2)                              2020-02-14T14:09:06+0000
Keys:  Help   Display mode   Restart statistics   Order of fields   quit
                                       Packets               Pings
 Host                                Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. 172.19.0.1                        0.0%    21    0.0   0.0   0.0   0.1   0.0
 2. (waiting for reply)
 3. 11.220.36.13                      0.0%    20    4.1   3.6   2.5   7.7   1.1
 4. 11.220.36.54                     10.0%    20    5.4  14.0   4.5  40.4   9.3
 5. 10.54.139.233                     0.0%    20    2.0   1.9   1.6   6.0   1.0
 6. 117.49.35.186                     0.0%    20    3.9   4.2   2.2  24.9   5.1
 7. 116.251.113.161                   0.0%    20    3.1   3.6   3.1   9.3   1.3
 8. 183.61.45.13                      0.0%    20    3.1   3.1   3.1   3.2   0.0
 9. 58.61.162.149                     0.0%    20    3.9   8.1   3.8  35.3   7.9
10. 119.147.219.249                   0.0%    20    8.0   9.9   6.6  13.6   2.5
11. 202.97.91.30                     10.0%    20   22.7  22.6  20.7  24.5   1.2
12. 202.97.43.229                    21.1%    20  123.9 123.2 108.3 125.9   4.3
13. 202.97.89.54                      0.0%    20   45.9  44.0  39.7  55.5   3.5
14. 202.97.62.214                     5.0%    20   58.3  59.9  52.5  65.3   3.4
15. 108.170.241.33                   10.0%    20   12.8  12.6  12.4  12.9   0.1
16. 216.239.42.89                     0.0%    20   36.5  40.8  33.3  46.4   4.1
17. dns.google                        5.0%    20   34.1  39.7  31.5  46.6   4.5

```

After:

```bash
                             My traceroute  [v0.93]
ec1d9d99af4c (172.19.0.2)                              2020-02-14T14:12:16+0000
Keys:  Help   Display mode   Restart statistics   Order of fields   quit
                                       Packets               Pings
 Host                                Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. 172.19.0.1                       94.4%    19    0.1   0.1   0.1   0.1   0.0
 2. 10.0.0.1                          0.0%    19    1.4   1.4   1.3   1.4   0.0
 3. (waiting for reply)
 4. 11.220.36.13                      0.0%    19    3.2   3.9   2.7  10.6   1.7
 5. 11.220.36.138                     5.3%    19   12.9  19.2   4.2  57.9  13.8
 6. 11.217.38.254                     0.0%    19    1.9   2.0   1.8   3.6   0.5
 7. 119.38.215.138                    0.0%    19    2.6   5.4   2.3  10.5   3.0
 8. 42.120.242.217                    0.0%    19    3.5   3.5   3.3   4.3   0.2
 9. 183.61.45.9                       0.0%    19    4.4   4.5   4.1   6.5   0.5
10. 58.61.162.133                     0.0%    19    6.6  14.5   4.7  57.0  17.2
11. 119.147.222.13                    0.0%    19    6.3  10.0   6.3  15.6   2.8
12. 202.97.91.130                    15.8%    19   22.2  23.1  21.2  25.4   1.1
13. 202.97.43.245                    21.1%    19   37.9  37.0  35.3  38.9   1.2
14. 202.97.89.54                      0.0%    18  130.9 130.8 121.4 134.8   3.4
15. 202.97.62.214                    16.7%    18   11.5  11.6  11.4  12.4   0.3
16. 108.170.241.97                    0.0%    18   12.7  12.8  12.5  13.0   0.1
17. 216.239.40.31                     5.6%    18   58.2  62.5  53.8  67.8   3.9
18. dns.google                        5.6%    18   12.3  12.2  12.2  12.3   0.0
```

Be care of the `10.0.0.1` hop.

If we want to bring the VPN up on boot, we need to create `/etc/network/interfaces.d/wg0` with encoded commands:

```bash
auto wg0
iface wg0 inet manual
pre-up ip link add dev wg0 type wireguard
pre-up wg setconf wg0 /etc/wireguard/wg0.conf
pre-up ip address add 10.0.0.2/24 dev wg0
up ip link set up dev wg0
post-up /bin/bash -c "printf 'nameserver %s\n' '10.0.0.1' | resolvconf -a tun.wg0 -m 0 -x"
post-up ip rule add from 172.19.0.0/16 table 10
post-up ip route add default via 10.0.0.2 table 10
post-up sysctl -w net.ipv4.conf.all.rp_filter=2 # 0 is ok
post-down ip link del dev wg0
```

Finished!