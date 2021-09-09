<!-- AIR:tour -->

# Segment Routing Demo

Description:

This demo will create a Relaxed LSR path for host1 to ping host2 across five Cumulus Linux devices. This demo uses the same layout and schema as the Cumulus docs. The diagram below links directly to the topology in the Cumulus docs.

### Network Diagram:

![Network Diagram](https://raw.githubusercontent.com/CumulusNetworks/docs/stage/static/images/cumulus-linux/segment-routing-example.png)

<!-- AIR:tour -->

### Troubleshooting

Helpful FRR and MPLS troubleshooting commands:

- show ip bgp summary
- show ip route
- show mpls status
- show mpls table
- show mpls fec

Helpful Linux troubleshooting commands:

- ip r s

### Segment Routing Troubleshooting

1. sudo vtysh -c "show ip bgp summary"

```
cumulus@r5:mgmt:~$ sudo vtysh -c "show ip bgp summary"

IPv4 Labeled Unicast Summary:
BGP router identifier 10.1.1.5, local AS number 65555 vrf-id 0
BGP table version 0
RIB entries 0, using 0 bytes of memory
Peers 3, using 64 KiB of memory
Peer groups 1, using 64 bytes of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt
r4(swp1)        4      65444       573       576        0    0    0 00:28:17            3        5
r2(swp2)        4      65222       574       576        0    0    0 00:28:17            4        5
r3(swp3)        4      65333       575       576        0    0    0 00:28:17            3        5

Total number of neighbors 3
```

One will see the labels that have been shared in the "IPv4 Labeled Unicast" table.

2. Show the MPLS Forwarding Equivalency Classes:

sudo vtysh -c "show mpls fec"

```
10.1.1.1/32
  Label: 101, Label Index: 1
  Client list: bgp(fd 42)
10.1.1.2/32
  Label: 102, Label Index: 2
  Client list: bgp(fd 42)
10.1.1.3/32
  Label: 103, Label Index: 3
  Client list: bgp(fd 42)
10.1.1.4/32
  Label: 104, Label Index: 4
  Client list: bgp(fd 42)
```

3. Pings between LSRs:

```
cumulus@r5:mgmt:~$ ping 10.1.1.1
vrf-wrapper.sh: switching to vrf "default"; use '--no-vrf-switch' to disable
PING 10.1.1.1 (10.1.1.1) 56(84) bytes of data.
64 bytes from 10.1.1.1: icmp_seq=1 ttl=63 time=0.700 ms
64 bytes from 10.1.1.1: icmp_seq=2 ttl=63 time=0.770 ms
64 bytes from 10.1.1.1: icmp_seq=3 ttl=63 time=0.755 ms
```

In this demo, you will be able to ping all of the loopback addresses from any LSR.

4. LSR routing information:

sudo vtysh -c "show ip route"

```
cumulus@r5:mgmt:~$ sudo vtysh -c "show ip route"
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure
B>* 10.1.1.1/32 [20/0] via fe80::4638:39ff:fe00:5, swp2, label 101, weight 1, 00:06:46
B>* 10.1.1.2/32 [20/0] via fe80::4638:39ff:fe00:5, swp2, label implicit-null, weight 1, 00:06:46
B>* 10.1.1.3/32 [20/0] via fe80::4638:39ff:fe00:9, swp3, label implicit-null, weight 1, 00:06:46
B>* 10.1.1.4/32 [20/0] via fe80::4638:39ff:fe00:b, swp1, label implicit-null, weight 1, 00:29:15
C>* 10.1.1.5/32 is directly connected, lo, 00:06:46
```

The "implicit-null" labels indicate that this route is one hop away from the current router. This means that the current router is the Penultimate Hop Popping (PHP) router

5. Server-based verification:

"ip r s"

```
cumulus@host1:~$ ip r s
default via 192.168.200.1 dev eth0
192.168.11.0/24 dev eth1 proto kernel scope link src 192.168.11.111
192.168.22.222  encap mpls  103 via 192.168.11.1 dev eth1
192.168.200.0/24 dev eth0 proto kernel scope link src 192.168.200.31
```

This will show what is currently installed within the kernel of the host.

6. Verification

Log into host2 and ping host1:

```
cumulus@host2:~$ ping 192.168.11.111 -c 4
PING 192.168.11.111 (192.168.11.111) 56(84) bytes of data.
64 bytes from 192.168.11.111: icmp_seq=1 ttl=61 time=1.55 ms
64 bytes from 192.168.11.111: icmp_seq=2 ttl=61 time=1.62 ms
64 bytes from 192.168.11.111: icmp_seq=3 ttl=61 time=1.69 ms
64 bytes from 192.168.11.111: icmp_seq=4 ttl=61 time=1.57 ms
```

### Errata

Remove Relaxed LSR routes on host1:

```
sudo ip route del 192.168.22.222/32 encap mpls 103 via inet 192.168.11.1
```

Remove Relaxed LSR routes on host2:

```
sudo ip route add 192.168.11.111/32 encap mpls 101 via inet 192.168.22.1
```

The above need to be removed before executing the playbook again.
