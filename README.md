# Segment Routing Demo

Description:

This demo will use the cldemo2 topology and create a Relaxed LSR path for server01 to ping server04 and a Strict LSR path for server02 to ping server05.

### Network Diagram:

![Network Diagram](https://gitlab.com/nvidia-networking/systems-engineering/poc-support/segment-routing-demo/-/tree/master/documentation/cldemo2-sr.png)


1. First, create a "Cumulus In the Cloud" Reference Topology within Cumulus Air. We will be presenting this demo using a subset of the overall cldemo2 topology.


2. Next, copy the gitlab repo unto the oob-mgmt-server:

    ```
    git clone https://gitlab.com/nvidia-networking/systems-engineering/poc-support/segment-routing-demo
    ```

3. Change directories to the following

    ```
    cd segment-routing-demo
    ```

4. Run the following:

    ```
    ansible-playbook cumulus-segment-routing.yml
    ```

This will setup an MPLS BGP overlay between the leaf01-04 and spine01-02. The remaining links in the topology to these elements will be disabled to avoid any issues.

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
IPv4 Labeled Unicast Summary:
BGP router identifier 10.1.1.1, local AS number 65111 vrf-id 0
BGP table version 0
RIB entries 0, using 0 bytes of memory
Peers 4, using 85 KiB of memory
Peer groups 1, using 64 bytes of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
leaf01(swp1)    4      65011     26258     26256        0    0    0 21:52:23            1
leaf02(swp2)    4      65012     26258     26255        0    0    0 21:52:23            1
leaf03(swp3)    4      65013     26258     26255        0    0    0 21:52:23            1
leaf04(swp4)    4      65014     26258     26254        0    0    0 21:52:23            1

Total number of neighbors 4
```

One will see the labels that have been shared in the "IPv4 Labeled Unicast" table.

2. Show the MPLS Forwarding Equivalency Classes:

sudo vtysh -c "show mpls fec"

```
10.1.1.11/32
  Label: 111, Label Index: 11
  Client list: bgp(fd 23)
10.1.1.12/32
  Label: 112, Label Index: 12
  Client list: bgp(fd 23)
10.1.1.13/32
  Label: 113, Label Index: 13
  Client list: bgp(fd 23)
10.1.1.14/32
  Label: 114, Label Index: 14
  Client list: bgp(fd 23)
```

3. Pings between LSRs:

```
cumulus@spine01:mgmt:~$ ping 10.1.1.11
vrf-wrapper.sh: switching to vrf "default"; use '--no-vrf-switch' to disable
PING 10.1.1.11 (10.1.1.11) 56(84) bytes of data.
64 bytes from 10.1.1.11: icmp_seq=1 ttl=64 time=0.668 ms
64 bytes from 10.1.1.11: icmp_seq=2 ttl=64 time=0.655 ms
64 bytes from 10.1.1.11: icmp_seq=3 ttl=64 time=0.623 ms
```

In this demo, you will be able to ping all of the loopback addresses from any LSR.

4. LSR routing information:

sudo vtysh -c "show ip route"

```
cumulus@spine01:mgmt:~$ sudo vtysh -c "show ip route"
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued route, r - rejected route

C>* 10.1.1.1/32 is directly connected, lo, 21:53:34
B>* 10.1.1.11/32 [20/0] via fe80::40f6:3dff:fef8:3b4f, swp1, label implicit-null, weight 1, 21:53:31
B>* 10.1.1.12/32 [20/0] via fe80::d869:afff:fe85:991d, swp2, label implicit-null, weight 1, 21:53:31
B>* 10.1.1.13/32 [20/0] via fe80::a2cc:bdff:feaf:d0b3, swp3, label implicit-null, weight 1, 21:53:31
B>* 10.1.1.14/32 [20/0] via fe80::c44a:4dff:fe1d:e4fd, swp4, label implicit-null, weight 1, 21:53:31
```

The "implicit-null" labels indicate that this route is one hop away from the current router. This means that the current router is the Penultimate Hop Popping (PHP) router

5. Server-based verification:

"ip r s"

```
default via 192.168.200.1 dev eth0
default via 192.168.200.1 dev eth0 proto dhcp src 192.168.200.31 metric 100
192.168.11.0/24 dev eth1 proto kernel scope link src 192.168.11.111
192.168.44.111  encap mpls  113 via 192.168.11.1 dev eth1
192.168.200.0/24 dev eth0 proto kernel scope link src 192.168.200.31
192.168.200.1 dev eth0 proto dhcp scope link src 192.168.200.31 metric 100
```

This will show what is currently installed within the kernel of the host.

6. Strict LSR Forwarding - server02 and server04

Running the "ip r s" command on server02 will show the following:

```
default via 192.168.200.1 dev eth0
default via 192.168.200.1 dev eth0 proto dhcp src 192.168.200.32 metric 100
192.168.22.0/24 dev eth2 proto kernel scope link src 192.168.22.222
192.168.55.222  encap mpls  101/113/102/114 via 192.168.22.1 dev eth2
192.168.200.0/24 dev eth0 proto kernel scope link src 192.168.200.32
192.168.200.1 dev eth0 proto dhcp scope link src 192.168.200.32 metric 100
```

On server02, the following is installed in the kernel:
"192.168.55.222  encap mpls  101/113/102/114 via 192.168.22.1 dev eth2"

The "101/113/102/114" is the labeled switch path, or the routing segment, that the traffic will take

7. Verification

Log into server01 and ping server04:

```
cumulus@server01:~$ ping 192.168.44.111
PING 192.168.44.111 (192.168.44.111) 56(84) bytes of data.
64 bytes from 192.168.44.111: icmp_seq=1 ttl=61 time=3.09 ms
64 bytes from 192.168.44.111: icmp_seq=2 ttl=61 time=2.36 ms
64 bytes from 192.168.44.111: icmp_seq=3 ttl=61 time=2.79 ms
```

Log into server02 and ping server05:

```
cumulus@server02:~$ ping 192.168.55.222
PING 192.168.55.222 (192.168.55.222) 56(84) bytes of data.
64 bytes from 192.168.55.222: icmp_seq=1 ttl=62 time=4.86 ms
64 bytes from 192.168.55.222: icmp_seq=2 ttl=62 time=3.60 ms
64 bytes from 192.168.55.222: icmp_seq=3 ttl=62 time=4.57 ms
```

### Errata

Remove Strict LSR routes:

```
ip route del 192.168.55.222/32 encap mpls 101/113/102/114 via inet 192.168.22.1
```

Remove Relaxed LSR routes:
```
ip route del 192.168.44.111/32 encap mpls 113 via inet 192.168.11.1
```
