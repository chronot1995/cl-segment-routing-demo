# Segment Routing Demo

=== WORK IN PROGRESS ===

The first thing is to create a "Cumulus In the Cloud" Reference Topology within Cumulus Air. We will be presenting this demo using a subset of the overall cldemo2 topology.


2. Copy the Git repo unto the oob-mgmt-server:

    ```
    git clone https://github.com/chronot1995/cumulus-segment-routing
    ```

3. Change directories to the following

    ```
    cumulus-segment-routing/automation
    ```

4. Run the following:

    ```
    ./provision.sh
    ```

This will run the automation script and configure the environment.

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

```IPv4 Labeled Unicast Summary:
BGP router identifier 10.1.1.1, local AS number 65111 vrf-id 0
BGP table version 0
RIB entries 0, using 0 bytes of memory
Peers 2, using 39 KiB of memory
Peer groups 1, using 64 bytes of memory

Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
r2(swp2)        4      65222     599     598        0    0    0 00:29:27            3
r4(swp4)        4      65444     596     597        0    0    0 00:29:26            1
```

One will see the labels that have been shared in the "IPv4 Labeled Unicast" table.

2. Show the MPLS Forwarding Equivalency Classes:

sudo vtysh -c "show mpls fec"

```
10.1.1.2/32
  Label: 102, Label Index: 2
  Client list: bgp(fd 16)
10.1.1.3/32
  Label: 103, Label Index: 3
  Client list: bgp(fd 16)
10.1.1.4/32
  Label: 104, Label Index: 4
  Client list: bgp(fd 16)
10.1.1.5/32
  Label: 105, Label Index: 5
  Client list: bgp(fd 16)
```

3. Pings between LSRs:

```
r1# ping 10.1.1.5
PING 10.1.1.5 (10.1.1.5) 56(84) bytes of data.
64 bytes from 10.1.1.5: icmp_seq=1 ttl=63 time=1.57 ms
64 bytes from 10.1.1.5: icmp_seq=2 ttl=63 time=1.82 ms
```

In this demo, you will be able to ping all of the loopback addresses from any LSR.

4. LSR routing information:

sudo vtysh -c "show ip route"

```
C>* 10.1.1.1/32 is directly connected, lo, 00:32:51
B>* 10.1.1.2/32 [20/0] via fe80::4638:39ff:fe00:2, swp2, label implicit-null, 00:32:49
B>* 10.1.1.3/32 [20/0] via fe80::4638:39ff:fe00:2, swp2, label 103, 00:32:48
B>* 10.1.1.4/32 [20/0] via fe80::4638:39ff:fe00:4, swp4, label implicit-null, 00:32:48
B>* 10.1.1.5/32 [20/0] via fe80::4638:39ff:fe00:2, swp2, label 105, 00:32:48
C>* 192.168.11.0/24 is directly connected, swp10, 00:32:51
```

The "implicit-null" labels indicate that this route is one hop away from the current router. This means that the current router is the Penultimate Hop Popping (PHP) router

5. Host-based verification:

"ip r s"

```
default via 10.0.2.2 dev vagrant
10.0.2.0/24 dev vagrant proto kernel scope link src 10.0.2.15
192.168.11.0/24 dev eth1 proto kernel scope link src 192.168.11.111
192.168.22.222  encap mpls 103 via 192.168.11.1 dev eth1
192.168.200.0/24 dev eth0 proto kernel scope link src 192.168.200.11
```

This will show what is currently installed within the kernel of the host.

6. Strict LSR Forwarding

The configuration example in this demo will take a relaxed label path in order to reach the destination. There is also the option of providing a strict path of LSRs to traverse.

We will remove the relaxed configuration on host1 with the following:

```
sudo ip route del 192.168.22.222/32 encap mpls 103 via inet 192.168.11.1
```

Now, we will force traffic to go through r4, r5, and then to r3 with the following:

```
sudo ip route add 192.168.22.222/32 encap mpls 104/105/103 via inet 192.168.11.1
```

We now see the following in the kernel:

"ip r s"

```
default via 10.0.2.2 dev vagrant
10.0.2.0/24 dev vagrant proto kernel scope link src 10.0.2.15
192.168.11.0/24 dev eth1 proto kernel scope link src 192.168.11.111
192.168.22.222  encap mpls  104/105/103 via 192.168.11.1 dev eth1
192.168.200.0/24 dev eth0 proto kernel scope link src 192.168.200.11
```

To execute the above commands strict LSR configuration, run the following playbook from the automation directory:

```
ansible-playbook cumulus-segment-routing-strict.yml
```

To reset the demo back to a relaxed LSR configuration, run the following playbook from the automation directory:

```
ansible-playbook cumulus-segment-routing-relaxed.yml
```


### Errata
=======
