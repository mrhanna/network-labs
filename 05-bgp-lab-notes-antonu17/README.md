# BGP Lab Notes (antonu17)

When I was studying for my CCNA, where BGP was concerned, the curriculum basically taught me:

1. it's an **external routing protocol** of the **path-vector** type,
2. it has powered the internet since ~1994,
3. it's complex and beyond the scope of the CCNA.

The suspense has been killing me, so I decided to learn what it's about.

First, I used APNIC Academy's [Introduction to BGP](https://academy.apnic.net/en/course/introduction-to-bgp) to get a handle on the basics. Then, I discovered a [GitHub repo](https://github.com/antonu17/lab-network-bgp) by Anton Ustyuzhanin that offers some premade configuration and troubleshooting tasks in a topology with three transit provider and two datacenter ASes; its description was:

> Ever wanted to learn BGP? Kickstart your journey with this interactive Lab and Practical Tasks!

The instructions said to document my process; the purpose of this README is to do so informally. Before this lab, I had never touched FRR or BIRD, so I was learning as I went.

## Topology

![Topology Diagram](https://raw.githubusercontent.com/antonu17/lab-bgp-anycast/refs/heads/main/diagram-details.drawio.svg)

_Graphic by [Anton Ustyuzhanin](https://github.com/antonu17), the creator of this lab. [[Source](https://github.com/antonu17/lab-network-bgp/blob/main/Topology.md)]_

## Task Index

- **Configuration Tasks**
  - [Task 1](#task-1)
  - [Task 2](#task-2)
  - [Task 3](#task-3)
  - [Task 4](#task-4)
  - [Task 5](#task-5)
  - [Task 6](#task-6)
  - [Task 7](#task-7)
- **Troubleshooting Tasks**
  - [Task 1](#task-1-1)
  - [Task 2](#task-2-1)
  - [Task 3](#task-3-1)
  - [Task 4](#task-4-1)
  - [Task 5](#task-5-1)
  - [Task 6](#task-6-1)
  - [Task 7](#task-7-1)
  - [Task 8](#task-8)

## Configuration Tasks

### Task 1

> Peering partners have reported concerns about excessive prefix announcements. Ensure no prefixes longer than /24 for IPv4 and /48 for IPv6 are advertised from both Data Centers. Maintain uninterrupted connectivity during this process. You may only modify the configuration of router1.dc1 and router1.dc2.

#### Process & Resolution

First, I want to see the problem for myself. `docker exec -it clab-bgp-router1.dc1 vtysh`. Poking around a bit, FRR/vtysh feels similar to Cisco IOS. Let's see what we're advertising to AS101:

```
router1.dc1# show bgp ipv4 neighbors 10.101.1.1 advertised-routes
BGP table version is 100, local router ID is 10.1.254.1, vrf id 0
Default local pref 100, local AS 1443
Status codes:  s suppressed, d damped, h history, u unsorted, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  1.1.1.0/24       0.0.0.0                                0 65100 65101 i
 *>  2.2.2.0/24       0.0.0.0                                0 65100 65101 i
 *>  10.1.1.0/30      0.0.0.0                                0 65100 i
 *>  10.1.2.0/30      0.0.0.0                                0 65100 i
 *>  10.1.3.0/30      0.0.0.0                                0 65100 i
 *>  10.1.4.0/30      0.0.0.0                                0 65100 i
 *>  10.1.5.0/30      0.0.0.0                                0 65100 i
 *>  10.1.6.0/30      0.0.0.0                                0 65100 i
 *>  10.1.7.0/30      0.0.0.0                                0 65100 i
 *>  10.1.8.0/30      0.0.0.0                  0             0 i
 *>  10.1.9.0/30      0.0.0.0                  0             0 i
 *>  10.1.254.1/32    0.0.0.0                  0             0 i
 *>  10.1.254.2/32    0.0.0.0                                0 65100 i
 *>  10.1.254.3/32    0.0.0.0                                0 65100 i
 *>  10.1.255.1/32    0.0.0.0                                0 65100 65101 i
 *>  10.1.255.2/32    0.0.0.0                                0 65100 65101 i
 *>  10.1.255.3/32    0.0.0.0                                0 65100 65101 i
 *>  111.11.1.0/24    0.0.0.0                                0 65100 65101 i
 *>  111.11.2.0/24    0.0.0.0                                0 65100 65101 i
 *>  111.11.3.0/24    0.0.0.0                                0 65100 65101 i

Total number of prefixes 20, total number of paths 20
```

Can confirm, every loopback and transit link in the data center is being advertised out! IPv6 is behaving similarly.

Next, I poked around the running config (vtysh doesn't support `| section`; that's annoying): the lab designer assigned the neighbor links to a peer group `TRANS` which is assigned to a route map that advertises all routes; however, the two neighbor IPs are assigned to a more specific route map that I suspect is _supposed to be_ in charge of limiting the routes to be advertised.

```
router bgp 1443
 bgp router-id 10.1.254.1
 ...
 neighbor TRANS peer-group
 ...
 neighbor 10.101.1.1 peer-group TRANS
 neighbor 10.101.1.1 remote-as 101
 neighbor 10.102.1.1 peer-group TRANS
 neighbor 10.102.1.1 remote-as 102
 !
 address-family ipv4 unicast
  ...
  neighbor TRANS route-map ALLOW_ALL in
  neighbor TRANS route-map ALLOW_ALL out
  neighbor 10.101.1.1 route-map AS101_OUT out
  neighbor 10.102.1.1 route-map AS102_IN in
  neighbor 10.102.1.1 route-map AS102_OUT out
 exit-address-family
```

So let's take a look at AS101_OUT and AS102_OUT and associated prefix-lists:

```
!
ip prefix-list DC_SUBNETS seq 5 permit 10.1.0.0/16 le 32
ip prefix-list DC_SUBNETS seq 10 permit 111.11.0.0/16 le 32
ip prefix-list DC_SUBNETS seq 15 permit 1.1.1.0/24 le 32
ip prefix-list DC_SUBNETS seq 20 permit 2.2.2.0/24 le 32
ip prefix-list DC_SUBNETS seq 25 deny 0.0.0.0/0 le 32
!
ipv6 prefix-list DC_SUBNETS seq 5 permit fc00:dc1::/32 le 128
ipv6 prefix-list DC_SUBNETS seq 10 permit fc00:11::/32 le 128
ipv6 prefix-list DC_SUBNETS seq 15 permit fc00:aa::/32 le 128
ipv6 prefix-list DC_SUBNETS seq 20 deny ::/0 le 128
!
route-map ALLOW_ALL permit 10
exit
!
route-map AS101_OUT permit 10
 match ip address prefix-list DC_SUBNETS
exit
!
route-map AS101_OUT permit 20
 match ipv6 address prefix-list DC_SUBNETS
exit
!
route-map AS102_IN permit 10
 set local-preference 80
exit
!
route-map AS102_OUT permit 10
 match ip address prefix-list DC_SUBNETS
 set as-path prepend 1443 1443
exit
!
route-map AS102_OUT permit 20
 match ipv6 address prefix-list DC_SUBNETS
 set as-path prepend 1443 1443
exit
```

The topology diagram says that AS102 is the less preferred link; here I can see that this was accomplished by AS-path prepending out to AS102, and assigning a lower local preference in from AS102.

**But more to the task at hand,** I see that they're all using the same prefix-list, and the prefix lists are configured much too broadly--they're explicitly advertising subnets up to /32 for IPv4 and /128 for IPv6. Let's fix that by tightening up those prefix-lists, and clear the process with `soft out` so that the connection doesn't flap.

```
router1.dc1# conf t
router1.dc1(config)# ip prefix-list DC_SUBNETS seq 5 permit 10.1.0.0/16 le 24
router1.dc1(config)# ip prefix-list DC_SUBNETS seq 10 permit 111.11.0.0/16 le 24
router1.dc1(config)# ip prefix-list DC_SUBNETS seq 15 permit 1.1.1.0/24
router1.dc1(config)# ip prefix-list DC_SUBNETS seq 20 permit 2.2.2.0/24
router1.dc1(config)# ipv6 prefix-list DC_SUBNETS seq 5 permit fc00:dc1::/32 le 48
router1.dc1(config)# ipv6 prefix-list DC_SUBNETS seq 10 permit fc00:11::/32 le 48
router1.dc1(config)# ipv6 prefix-list DC_SUBNETS seq 15 permit fc00:aa::/32 le 48
router1.dc1(config)# end
router1.dc1# clear bgp * soft out
```

And let's see if it worked:

```
router1.dc1# show bgp ipv4 neighbors 10.101.1.1 advertised-routes
 ...
     Network          Next Hop            Metric LocPrf Weight Path
 *>  1.1.1.0/24       0.0.0.0                                0 65100 65101 i
 *>  2.2.2.0/24       0.0.0.0                                0 65100 65101 i
 *>  111.11.1.0/24    0.0.0.0                                0 65100 65101 i
 *>  111.11.2.0/24    0.0.0.0                                0 65100 65101 i
 *>  111.11.3.0/24    0.0.0.0                                0 65100 65101 i

Total number of prefixes 5, total number of paths 5
```

Much better! `show bgp ipv6 neighbors 10.101.1.1 advertised-routes` shows that it's working on IPv6 as well. router1.dc2, so I repeated the process there as well.

~~_There's another issue I see that is adjacent to the task at hand: the topology diagram says that 111.11.x.x is for DC-local anycast, and that 10.1.x.x is for Infrastructure. Infra isn't getting advertised anymore since all the networks are /30 and up, but those DC-local routes still are. There's no reason for those to be in the prefix-lists at all, so I'm going to go ahead and remove those--`no ip prefix-list DC_SUBNETS seq 5` and so on._~~

_Actually, I got ahead of myself here, and had to undo the above, since this is relevant to tasks 5 and 6._

[Back to Task Index](#task-index)

---

### Task 2

> The Security team has raised concerns about BGP advertisements containing private Autonomous System Numbers (ASNs) in the range 64512 to 65534 within the AS PATH. Exclude these ASNs from all advertised routes. Configuration changes are restricted to router1.dc1 and router1.dc2.

#### Process & Resolution

First, let's verify the problem:

```
router1.dc2# show bgp peer TRANS

BGP peer-group TRANS
  Peer-group type is external
  Configured address-families: IPv4 Unicast; IPv6 Unicast;
  Peer-group members:
    10.103.1.1  Established
    10.102.6.1  Established

router1.dc2# show bgp ipv4 nei 10.103.1.1 ad
...
     Network          Next Hop            Metric LocPrf Weight Path
 *>  1.1.1.0/24       0.0.0.0                                0 65200 65201 i
 *>  2.2.2.0/24       0.0.0.0                                0 65200 65201 i

router1.dc2# show bgp ipv4 nei 10.102.6.1 ad
...
     Network          Next Hop            Metric LocPrf Weight Path
 *>  1.1.1.0/24       0.0.0.0                                0 1443 1443 65200 65201 i
 *>  2.2.2.0/24       0.0.0.0                                0 1443 1443 65200 65201 i
```

Yes, the AS-paths have private ASNs (>= 64512). A quick google search shows that there is a neighbor option `remove-private-AS`--maybe this will be an (almost) one-liner?

```
router1.dc2# conf t
router1.dc2(config)# router bgp 1443
router1.dc2(config-router)# address-family ipv4
router1.dc2(config-router-af)# neighbor TRANS remove-private-AS
router1.dc2(config-router-af)# address-family ipv6
router1.dc2(config-router-af)# neighbor TRANS remove-private-AS
router1.dc2(config-router-af)# end
router1.dc2# show bgp ipv4 nei 10.103.1.1 ad
...
     Network          Next Hop            Metric LocPrf Weight Path
 *>  1.1.1.0/24       0.0.0.0                                0 i
 *>  2.2.2.0/24       0.0.0.0                                0 i

router1.dc2# show bgp ipv4 nei 10.102.6.1 ad
...
     Network          Next Hop            Metric LocPrf Weight Path
 *>  1.1.1.0/24       0.0.0.0                                0 1443 1443 65200 65201 i
 *>  2.2.2.0/24       0.0.0.0                                0 1443 1443 65200 65201 i
```

Almost! By default, `remove-private-AS` stops removing private ASNs once it encounters a public ASN reading left-to-right. In this case, there actually isn't another public AS along the path, but I guess FRR is prepending its own local ASN before it evaluates `remove-private-AS`.

There's a solution to this: `remove-private-AS all`. This removes private ASNs regardless of where they occur in the path. This option comes with caveats, but none of them apply here--in this topology, there are no public ASes on the other side of my private ASes, and there never will be. So let's try again:

```
router1.dc2# conf t
router1.dc2(config)# router bgp 1443
router1.dc2(config-router)# address-family ipv4
router1.dc2(config-router-af)# neighbor TRANS remove-private-AS all
router1.dc2(config-router-af)# address-family ipv6
router1.dc2(config-router-af)# neighbor TRANS remove-private-AS all
router1.dc2(config-router-af)# end
router1.dc2# show bgp ipv4 nei 10.102.6.1 ad
...

     Network          Next Hop            Metric LocPrf Weight Path
 *>  1.1.1.0/24       0.0.0.0                                0 1443 1443 i
 *>  2.2.2.0/24       0.0.0.0                                0 1443 1443 i
```

Now our path is clean!

[Back to Task Index](#task-index)

---

### Task 3

> Currently, requests from AS 102 customers are distributed between two Data Centers. Configure routing so that Data Center #2 handles all requests to the network 2.2.2.0/24 from AS 102. Ensure that requests from AS 101 customers to the same network continue to be partially served by Data Center #1. Note: Modifications to transit provider configurations are not allowed.

#### Process

Without access to transit provider configuration, and since the lab transit providers don't have communities set up that allow me to request a local_pref, my best bet is probably just to do some prepending to the AS_paths. (MED almost could be an option here since both DCs share the same ASN, but since MED is non-transitive, it wouldn't be able to pass through all the transit providers. Besides [all the other caveats with MED](https://ine.com/blog/2011-10-12-understanding-bgp-med-and-bgp-deterministic-med)).

Studying the [topology diagram](https://raw.githubusercontent.com/antonu17/lab-bgp-anycast/refs/heads/main/diagram-details.drawio.svg), and based on information gathered earlier in task 1:

- AS 102 has two paths each to DC1 and DC2:
  - DC1
    - `101 1443`
    - `1443 1443 1443` (less preferred direct link)
  - DC2
    - `103 1443`
    - `1443 1443 1443` (less preferred direct link)
- AS 101 also has two paths each to DC1 and DC2
  - DC1
    - 1443
    - 102 1443 1443 1443
  - DC2
    - 102 103 1443
    - 102 1443 1443 1443

I can make DC2 preferred by making DC1 less preferred. Specifically, I can solve this by prepending twice in advertisements to 101 and four times (instead of the current two times) in advertisements to 102:

| Scenario             | Best path to DC1         | Best path to DC2 |
| -------------------- | ------------------------ | ---------------- |
| 101 and 103 are up   | 101 1443 1443 1443       | 103 1443         |
| 103 is down          | 101 1443 1443 1443       | 1443 1443 1443   |
| 101 is down          | 1443 1443 1443 1443 1443 | 103 1443         |
| 101 and 103 are down | 1443 1443 1443 1443 1443 | 1443 1443 1443   |

By doing four prepends to DC1 advertisements to 102 (the less preferred link), I still have it so the link via 101 would preferred over the direct link if DC2 were to become inaccessible.

The prompt also says _"Ensure that requests from AS 101 customers to the same network continue to be partially served by Data Center #1."_ This should be the case - AS101 direct to DC1 has the path `1443 1443 1443`, which ties with `102 103 1443`.

#### Resolution

This only needs to be applied to the 2.2.2.0/24 subnet. AFAIK, there isn't a clean way to do this additively in FRR (i.e. follow the existing process; then, if it's 2.2.2.0/24, prepend two more times). So I will need to make a prefix-list containing only 2.2.2.0/24, have my route maps match this list earlier than the currently used one, and apply all the prepends at once.

```
router1.dc1# conf t
router1.dc1(config)# ip prefix-list UNPREFERRED_SUBNETS seq 10 permit 2.2.2.0/24
router1.dc1(config)# route-map AS101_OUT permit 5
router1.dc1(config-route-map)# match ip address prefix-list UNPREFERRED_SUBNETS
router1.dc1(config-route-map)# set as-path prepend 1443 1443
router1.dc1(config-route-map)# exit
router1.dc1(config)# route-map AS102_OUT permit 5
router1.dc1(config-route-map)# match ip address prefix-list UNPREFERRED_SUBNETS
router1.dc1(config-route-map)# set as-path prepend 1443 1443 1443 1443
router1.dc1(config-route-map)# end
router1.dc1# clear bgp * soft out
```

For lab purposes, I can verify path selection in AS 101 and 102 by remoting into a router in each and checking paths.

```
router3.as102# show bgp ipv4 2.2.2.0/24
BGP routing table entry for 2.2.2.0/24, version 158
Paths: (1 available, best #1, table default)
  Not advertised to any peer
  103 1443
    10.102.5.1 from 10.102.5.1 (10.102.254.2)
      Origin IGP, localpref 100, valid, internal, best (First path received)
      Last update: Sat Sep 26 20:58:15 2026
router3.as102# show bgp ipv4 1.1.1.0/24
BGP routing table entry for 1.1.1.0/24, version 117
Paths: (2 available, best #1, table default)
  Not advertised to any peer
  101 1443
    10.102.4.1 from 10.102.4.1 (10.102.254.1)
      Origin IGP, localpref 100, valid, internal, multipath, best (Router ID)
      Last update: Sat Sep 26 19:25:52 2026
  103 1443
    10.102.5.1 from 10.102.5.1 (10.102.254.2)
      Origin IGP, localpref 100, valid, internal, multipath
      Last update: Sat Sep 26 19:05:30 2026

---

router1.as101# show bgp ipv4 2.2.2.0/24
BGP routing table entry for 2.2.2.0/24, version 123
Paths: (2 available, best #1, table default)
  Advertised to peers:
  10.101.1.2 10.102.2.1
  1443 1443 1443
    10.101.1.2 from 10.101.1.2 (10.1.254.1)
      Origin IGP, valid, external, best (Older Path)
      Last update: Sat Sep 26 20:58:25 2026
  102 103 1443
    10.102.2.1 from 10.102.2.1 (10.102.254.1)
      Origin IGP, valid, external
      Last update: Sat Sep 26 20:58:16 2026
```

Looks good. AS101 does not have multipath enabled and is preferring DC1 even though it has an equal route to DC2. Since I can't adjust configuration on transit infra per task instructions, I can't do anything about this.

I can also log into the "eyeball" in those ASes and curl those anycast IPs since those endpoints are running a webserver that prints their hostnames.

```
# In AS 102, curls to 1.1.1.1 should split between DC1 and DC2.

root@eyeball:/# for i in {1..10}; do curl --local-port 50000-60000 -s http://1.1.1.1/ ; done
server1.dc2
server1.dc2
server3.dc1
server2.dc1
server2.dc1
server3.dc1
server3.dc1
server3.dc1
server1.dc2
server2.dc1

# In AS 102, curls to 2.2.2.2 should all go to DC2.

root@eyeball:/# for i in {1..10}; do curl --local-port 50000-60000 -s http://2.2.2.2/ ; done
server2.dc2
server2.dc2
server1.dc2
server1.dc2
server3.dc2
server3.dc2
server1.dc2
server2.dc2
server3.dc2
server1.dc2

# In AS 101, curls to 1.1.1.1 should all go to DC1.

root@eyeball:/# for i in {1..10}; do curl --local-port 50000-60000 -s http://1.1.1.1/ ; done
server2.dc1
server3.dc1
server1.dc1
server2.dc1
server2.dc1
server2.dc1
server3.dc1
server3.dc1
server1.dc1
server3.dc1

# In AS 101, curls to 2.2.2.2 should split between DC1 and DC2.
# But they don't, because multipath isn't enabled upstream.

root@eyeball:/# for i in {1..10}; do curl --local-port 50000-60000 -s http://2.2.2.2/ ; done
server1.dc1
server3.dc1
server2.dc1
server1.dc1
server3.dc1
server1.dc1
server1.dc1
server3.dc1
server1.dc1
server1.dc1
```

I'll call it a win!

[Back to Task Index](#task-index)

---

### Task 4

> As a network engineer in AS 102, configure routing so that all traffic destined for 1.1.1.0/24 prefers the peering partner AS 103. If AS 103 becomes unavailable, ensure AS 101 acts as a backup route. Configuration changes are restricted to routers within AS 102.

#### Process & Resolution

This is a good use case for `LOCAL_PREF`. The default value is `100`, and higher wins, so I will increase the preference of the route via AS 103 a bit, to `110`. Because the instructions say to _ensure_ that AS 101 acts as a backup route, I will also increase the LOCAL_PREF of the route via AS 101 a lesser bit, to `105`--in the lab's current configuration, this isn't strictly necessary, but this will ensure that the route via AS 101 remains preferred to the direct links if either DC should change the priority on their side via AS_PATH prepending, etc.

I will make the change to LOCAL_PREF on the router that receives the route, that is, router2.as102 for AS 103, and router1.as102 for AS 101. I will use a similar procedure to the last task, adding a prefix list and modifying or creating route maps.

First, I need to see how the IPv4 AF is configured to use route maps to begin with:

```
router2.as102# show run
...
 !
 address-family ipv4 unicast
  neighbor LOCAL soft-reconfiguration inbound
  neighbor LOCAL route-map ALLOW_ALL in
  neighbor LOCAL route-map ALLOW_ALL out
  neighbor TRANS soft-reconfiguration inbound
  neighbor TRANS route-map ALLOW_ALL in
  neighbor TRANS route-map TRANSIT_AGGR out
...
```

The two route maps seen above are the only two globally configured on the router. So I will need to make a new route map that matches 1.1.1.0/24 and applies LOCAL_PREF, and then lets everything else in; then apply that to the relevant neighbor.

```
router2.as102# conf t
router2.as102(config)# ip prefix-list PREFERRED_SUBNETS seq 10 permit 1.1.1.0/24
router2.as102(config)# route-map AS103_IN permit 10
router2.as102(config-route-map)# match ip address prefix-list PREFERRED_SUBNETS
router2.as102(config-route-map)# set local-preference 110
router2.as102(config-route-map)# exit
router2.as102(config)# route-map AS103_IN permit 20
router2.as102(config-route-map)# exit
router2.as102(config)# router bgp 102
router2.as102(config-router)# address-family ipv4 unicast
router2.as102(config-router-af)# neighbor 10.102.7.2 route-map AS103_IN in
router2.as102(config-router-af)# end

!-------

router1.as102# conf t
router1.as102(config)# ip prefix-list PREFERRED_SUBNETS seq 10 permit 1.1.1.0/24
router1.as102(config)# route-map AS101_IN permit 10
router1.as102(config-route-map)# match ip address prefix-list PREFERRED_SUBNETS
router1.as102(config-route-map)# set local-preference 105
router1.as102(config-route-map)# exit
router1.as102(config)# route-map AS101_IN permit 20
router1.as102(config-route-map)# exit
router1.as102(config)# router bgp 102
router1.as102(config-router)# address-family ipv4 unicast
router1.as102(config-router-af)# neighbor 10.102.2.2 route-map AS101_IN in
router1.as102(config-router-af)# end
```

And finally, a `show` command to verify the new preferences:

```
router1.as102# show bgp ipv4 1.1.1.0/24
BGP routing table entry for 1.1.1.0/24, version 136
Paths: (3 available, best #1, table default)
  Advertised to peers:
  10.102.1.2 10.102.2.2
  103 1443
    10.102.3.2 from 10.102.3.2 (10.102.254.2)
      Origin IGP, localpref 110, valid, internal, best (Local Pref)
      Last update: Mon Sep 28 18:38:14 2026
  101 1443
    10.102.2.2 from 10.102.2.2 (10.101.254.1)
      Origin IGP, localpref 105, valid, external
      Last update: Mon Sep 28 18:41:43 2026
  1443 1443 1443
    10.102.1.2 from 10.102.1.2 (10.1.254.1)
      Origin IGP, valid, external
      Last update: Sat Sep 26 20:58:25 2026
```

Success!

[Back to Task Index](#task-index)

---

### Task 5

> Data Center #2 is experiencing high traffic. Configure server3.dc2 to handle all incoming traffic directed toward the DC-local anycast range 222.22.1.0/24 originating from outside this DC. You are only allowed to modify the configuration of server3.dc2.

#### Process & Resolution

This one tripped me up quite a bit! It seems that none of my go-to BGP attributes will work here:

- **LOCAL_PREF** won't work because because the servers are not iBGP peers - they have no connection within the AS.
- **MED** was my first instinct since they are all in the same AS, and MED can be used to suggest to the next upstream AS which path to prefer. But with MED, the default is 0/unset, and I'm not allowed to access the other servers to set a higher MED. I also am not allowed to set `missing-as-worst` on the upstream routers. So this won't work either.
- **AS_PATH** prepending could only make me unprefer a path. No help since I can't access the other servers.
- **Communities** could be a clean solution but aren't implemented upstream. So this is no good either.

AFAIK, the only solution given the constraints is to advertise more specific, longer-prefixed routes so that they win by default. Currently, 222.22.1.0/24 is assigned to the loopback interface. I can add 222.22.1.0/25 and 222.22.1.128/25 to the interface and let BGP advertise those as well. The /25s won't make it outside the datacenter, but they don't need to--traffic will make it to the datacenter via the advertised /24s, and will get directed by the /25s on arrival. This feels like a hack, but it satisfies constraints.

The servers and routers in this part of the data center use BIRD instead of FRR, so I will be doing this config with iproute2.

```
root@server3:/# ip a add dev lo 222.22.1.0/25
root@server3:/# ip a add dev lo 222.22.1.128/25
root@server3:/# ip -br -d a
lo               UNKNOWN        127.0.0.1/8 10.2.255.3/32 222.22.3.0/24 222.22.1.0/24 1.1.1.0/24 2.2.2.0/24 222.22.1.0/25 222.22.1.128/25 fc00:aa::/48 fc00:22:1::/64 fc00:22:3::/64 fc00:dc2::255:3/128 ::1/128
eth2@if460       UP             10.2.6.2/30 fc00:dc2::6:2/126 fe80::a8c1:abff:feed:d434/64
eth1@if508       UP             10.2.5.2/30 fc00:dc2::5:2/126 fe80::a8c1:abff:fef9:3da5/64
```

Verify the routes are being advertised:

```
root@server3:/# birdc show route export router2
BIRD 2.0.12 ready.
Table master4:
222.22.1.128/25      unicast [direct1 20:36:19.357] * (240)
	dev lo
1.1.1.0/24           unicast [direct1 2026-09-26] * (240)
	dev lo
2.2.2.0/24           unicast [direct1 2026-09-26] * (240)
	dev lo
10.2.6.0/30          unicast [direct1 2026-09-26] * (240)
	dev eth2
10.2.5.0/30          unicast [direct1 2026-09-26] * (240)
	dev eth1
222.22.3.0/24        unicast [direct1 2026-09-26] * (240)
	dev lo
222.22.1.0/24        unicast [direct1 2026-09-26] * (240)
	dev lo
222.22.1.0/25        unicast [direct1 20:36:15.224] * (240)
	dev lo
10.2.255.3/32        unicast [direct1 2026-09-26] * (240)
	dev lo

Table master6:
fc00:dc2::255:3/128  unicast [direct1 2026-09-26] * (240)
	dev lo
fc00:aa::/48         unicast [direct1 2026-09-26] * (240)
	dev lo
fc00:dc2::6:0/126    unicast [direct1 2026-09-26] * (240)
	dev eth2
fc00:dc2::5:0/126    unicast [direct1 2026-09-26] * (240)
	dev eth1
fc00:22:3::/64       unicast [direct1 2026-09-26] * (240)
	dev lo
fc00:22:1::/64       unicast [direct1 2026-09-26] * (240)
	dev lo
```

Finally, I will do another curl check from AS 102 eyeball--requests to 222.22.3.1 should split between servers 2 and 3, but requests to 222.22.1.1 should all go to server 3.

```
root@eyeball:/# for i in {1..10}; do curl --local-port 50000-60000 -s http://222.22.3.1/ ; done
server2.dc2
server3.dc2
server2.dc2
server3.dc2
server3.dc2
server2.dc2
server3.dc2
server2.dc2
server2.dc2
server3.dc2
root@eyeball:/# for i in {1..10}; do curl --local-port 50000-60000 -s http://222.22.1.1/ ; done
server3.dc2
server3.dc2
server3.dc2
server3.dc2
server3.dc2
server3.dc2
server3.dc2
server3.dc2
server3.dc2
server3.dc2
```

Task complete!

[Back to Task Index](#task-index)

---

### Task 6

> The Security team has identified potential risks within the Data Center #1 DC-local anycast network 111.11.1.0/24. Ensure this network is inaccessible from outside the DC while maintaining internal connectivity. Note: Creating new firewall rules is prohibited.

This one I accidentally solved earlier in my ~~strikethrough~~ on Task 1--all I need to do is remove that network from the prefix-list that is advertised out of DC1's edge router.

```
router1.dc1# show run
...
!
ip prefix-list DC_SUBNETS seq 10 permit 111.11.0.0/16 le 24
ip prefix-list DC_SUBNETS seq 15 permit 1.1.1.0/24
ip prefix-list DC_SUBNETS seq 20 permit 2.2.2.0/24
ip prefix-list DC_SUBNETS seq 25 deny 0.0.0.0/0 le 32
ip prefix-list UNPREFERRED_SUBNETS seq 10 permit 2.2.2.0/24
...
router1.dc1# conf t
router1.dc1(config)# no ip prefix-list DC_SUBNETS seq 10
router1.dc1(config)# end
```

Verify that 111.11.1.0/24 is no longer being advertised:

```
router1.dc1# show bgp ipv4 neighbors 10.101.1.1 ad
...
     Network          Next Hop            Metric LocPrf Weight Path
 *>  1.1.1.0/24       0.0.0.0                                0 i
 *>  2.2.2.0/24       0.0.0.0                                0 1443 1443 i
```

And why not check pings from the eyeball in as102 while I'm at it?

```
root@eyeball:/# ping 222.22.1.1 -c 4
PING 222.22.1.1 (222.22.1.1) 56(84) bytes of data.
64 bytes from 222.22.1.1: icmp_seq=1 ttl=59 time=0.118 ms
64 bytes from 222.22.1.1: icmp_seq=2 ttl=59 time=0.075 ms
64 bytes from 222.22.1.1: icmp_seq=3 ttl=59 time=0.077 ms
64 bytes from 222.22.1.1: icmp_seq=4 ttl=59 time=0.068 ms

--- 222.22.1.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3049ms
rtt min/avg/max/mdev = 0.068/0.084/0.118/0.019 ms
root@eyeball:/# ping 111.11.1.1 -c 4
PING 111.11.1.1 (111.11.1.1) 56(84) bytes of data.

--- 111.11.1.1 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3069
```

_Honestly, that's not what I expected - I expected "unreachable." I logged into router3.as102 and saw it still had a default route through the containerlab management interface. For lab purposes this route probably shouldn't be in the same routing table, so I deleted it. Now the pings look the way I expected:_

```
root@eyeball:/# ping 111.11.1.1 -c 4
PING 111.11.1.1 (111.11.1.1) 56(84) bytes of data.
From 192.168.102.1 icmp_seq=1 Destination Net Unreachable
From 192.168.102.1 icmp_seq=2 Destination Net Unreachable
From 192.168.102.1 icmp_seq=3 Destination Net Unreachable
From 192.168.102.1 icmp_seq=4 Destination Net Unreachable
```

[Back to Task Index](#task-index)

---

### Task 7

> You are part of network traffic engineer team for Data Center #1. The planning team requested routing adjustments to optimize traffic distribution. Configure routing so that traffic destined for 1.1.1.0/24 enters the servers via router2.dc1, while traffic destined for 2.2.2.0/24 is directed through router3.dc1.

#### Process & Resolution

This seems like a straightforward place to use MED--there are two routers in the same AS trying to influence ingress routing from an adjacent AS non-transitively. It might be sufficient just to set MED for the subnet that the router _doesn't_ want (since lowest wins and default is 0), but since different platforms/configurations don't treat unset MED the same way, I will configure MED explicitly for both subnets on both routers.

These routers use BIRD instead of FRR. It took me a bit to get my bearings, but _wow_ is this configuration nicer than dealing with route maps and prefix lists!

```
# router2.dc1 - /etc/bird/bird.conf
# excerpt

protocol bgp router1 {
  local 10.1.8.2 as 65100;
  neighbor 10.1.8.1 as 1443;
  ipv4 {
    export filter {
      if net = 1.1.1.0/24 then {
        bgp_med = 5;
        accept;
      }

      if net = 2.2.2.0/24 then {
        bgp_med = 10;
        accept;
      }

      accept;
    };
    import all;
  };
  ipv6 {
    export all;
    import all;
  };
}
```

The same configuration on router3.dc1, but swapping the two `bgp_med` value assignments. Followed by running `birdc configure` on each.

From router1.dc1 (the edge router), I can verify that the correct path is being selected for these destinations:

- 1.1.1.0/24 should be routed through router2 (10.1.8.2)
- 2.2.2.0/24 should be routed through router3 (10.1.9.2)

```
router1.dc1# show bgp ipv4 1.1.1.0/24
BGP routing table entry for 1.1.1.0/24, version 276
Paths: (4 available, best #1, table default)
  Advertised to peers:
  10.1.8.2 10.1.9.2 10.101.1.1 10.102.1.1
  65100 65101
    10.1.8.2 from 10.1.8.2 (10.1.254.2)
      Origin IGP, metric 5, valid, external, best (MED)
      Last update: Tue Sep 29 02:45:12 2026
  65100 65101
    10.1.9.2 from 10.1.9.2 (10.1.254.3)
      Origin IGP, metric 10, valid, external
      Last update: Tue Sep 29 02:48:39 2026
  101 1443
    10.101.1.1 from 10.101.1.1 (10.101.254.1)
      Origin IGP, valid, external
      Last update: Sat Sep 26 19:25:52 2026
  102 103 1443
    10.102.1.1 from 10.102.1.1 (10.102.254.1)
      Origin IGP, localpref 80, valid, external
      Last update: Mon Sep 28 18:38:14 2026
router1.dc1# show bgp ipv4 2.2.2.0/24
BGP routing table entry for 2.2.2.0/24, version 277
Paths: (4 available, best #1, table default)
  Advertised to peers:
  10.1.8.2 10.1.9.2 10.101.1.1 10.102.1.1
  65100 65101
    10.1.9.2 from 10.1.9.2 (10.1.254.3)
      Origin IGP, metric 5, valid, external, best (MED)
      Last update: Tue Sep 29 02:48:39 2026
  65100 65101
    10.1.8.2 from 10.1.8.2 (10.1.254.2)
      Origin IGP, metric 10, valid, external
      Last update: Tue Sep 29 02:45:12 2026
  101 1443 1443 1443
    10.101.1.1 from 10.101.1.1 (10.101.254.1)
      Origin IGP, valid, external
      Last update: Sat Sep 26 20:58:15 2026
  102 103 1443
    10.102.1.1 from 10.102.1.1 (10.102.254.1)
      Origin IGP, localpref 80, valid, external
      Last update: Sat Sep 26 20:58:16 2026
```

All tasks complete!

[Back to Task Index](#task-index)

---

## Troubleshooting Tasks

_**N.B.** The [original repo](https://github.com/antonu17/lab-network-bgp/blob/main/Tasks.md#troubleshooting-tasks) contains troubleshooting tasks that introduce configuration errors by switching to the `tshoot` branch. It took me longer than I care to admit to realize that the BIRD configurations from above persisted through the branch switch since the `bird.conf`s are bind-mounted; as a result, some of the problems reported in the following tasks weren't actually happening. So `git reset --hard HEAD` before proceeding with the troubleshooting tasks._

### Task 1

> Identify and fix the issue stopping server1.dc2 from processing anycast requests.

#### Process & Resolution

Normally, I would start by verifying L1/L2 status--for lab purposes, I'm satisfied that it is online and reachable out of band.

First, I will check the interface addresses to make sure that the loopback interface has the anycast network. I will also check the BGP peering statuses.

```console
root@server1:/# ip -br -d a
lo               UNKNOWN        127.0.0.1/8 10.2.255.1/32 222.22.1.0/24 222.22.2.0/24 1.1.1.0/24 2.2.2.0/24 fc00:aa::/48 fc00:22:2::/64 fc00:22:1::/64 fc00:dc2::255:1/128 ::1/128
eth0@if592       UP             172.20.20.12/24 3fff:172:20:20::c/64 fe80::bcc7:50ff:fe6d:2b1/64
eth2@if567       UP             10.2.2.2/30 fc00:dc2::2:2/126 fe80::a8c1:abff:fe0b:8baf/64
eth1@if595       UP             10.2.1.2/30 fc00:dc2::1:2/126 fe80::a8c1:abff:fe0f:3d67/64
root@server1:/# birdc show protocols
BIRD 2.0.12 ready.
Name       Proto      Table      State  Since         Info
device1    Device     ---        up     16:26:10.339
direct1    Direct     ---        up     16:26:10.339
kernelv4   Kernel     master4    up     16:26:10.339
kernelv6   Kernel     master6    up     16:26:10.339
router2    BGP        ---        start  16:47:26.004  Idle          BGP Error: Bad peer AS
router3    BGP        ---        start  16:45:38.799  Idle          BGP Error: Bad peer AS
```

BGP is idle on both connections. It is most likely misconfigured. I will check the configuration for those two routers:

```console
root@server1:/# cat etc/bird/bird.conf
...
protocol bgp router2 {
        local 10.2.1.2 as 65201;
        neighbor 10.2.1.1 as 65000;
        allow local as;
        ipv4 {
                export all;
                import all;
        };
        ipv6 {
                export all;
                import all;
        };
}

protocol bgp router3 {
        local 10.2.2.2 as 65201;
        neighbor 10.2.2.1 as 65000;
        allow local as;
        ipv4 {
                export all;
                import all;
        };
        ipv6 {
                export all;
                import all;
        };
}
```

Per the topology diagram and the configuration on the other side, routers 2 and 3 belong in AS 65200, not 65000. Let's change those two numbers, reconfigure, and recheck peering status:

```console
root@server1:/# birdc configure
BIRD 2.0.12 ready.
Reading configuration from /etc/bird/bird.conf
Reconfigured
root@server1:/# birdc show protocols
BIRD 2.0.12 ready.
Name       Proto      Table      State  Since         Info
device1    Device     ---        up     16:26:10.339
direct1    Direct     ---        up     16:26:10.339
kernelv4   Kernel     master4    up     16:26:10.339
kernelv6   Kernel     master6    up     16:26:10.339
router2    BGP        ---        start  16:59:32.579  Active        Socket: Connection reset by peer
router3    BGP        ---        start  16:59:32.579  Active        Socket: Connection reset by peer

# a few seconds later, "active" will become "established"

root@server1:/# birdc show protocols
BIRD 2.0.12 ready.
Name       Proto      Table      State  Since         Info
device1    Device     ---        up     16:26:10.339
direct1    Direct     ---        up     16:26:10.339
kernelv4   Kernel     master4    up     16:26:10.339
kernelv6   Kernel     master6    up     16:26:10.339
router2    BGP        ---        up     17:01:03.481  Established
router3    BGP        ---        up     17:00:16.734  Established
```

It took some time for retry timers to expire, etc.; but now peering is established as it should be. server1.dc2 should be ready to serve anycast requests now.

_At this point, I entered an eyeball in AS103 to see if server1.dc2 would respond to 1.1.1.1. Traffic actually wasn't routed to DC2 at all, only DC1. I suspect that this may be a topic of a later task, so I'm going to leave it for now._

[Back to Task Index](#task-index)

---

### Task 2

> Customers in AS 102 are having trouble accessing the web service at IPv4 address 1.1.1.1, while the service works perfectly over IPv6 at fc00:aa::. Investigate the cause of the issue with IPv4 connectivity, identify the problem, and implement a fix. Use ping from eyeball.as102 to confirm your solution.

#### Process & Resolution

Sure enough, eyeball.as102 cannot ping 1.1.1.1. My next question is, does its gateway (i.e. router3.as102) have a route to 1.1.1.1?

```
router3.as102# show ip route 1.1.1.1
Routing entry for 1.1.1.0/24
  Known via "bgp", distance 200, metric 0, best
  Last update 02:25:38 ago
  Flags: Recursion iBGP Selected
  Status: Installed
  * 10.102.4.1, via eth1, weight 1

router3.as102# ping 1.1.1.1 count 2
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=60 time=0.119 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=60 time=0.094 ms

--- 1.1.1.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1011ms
rtt min/avg/max/mdev = 0.094/0.106/0.119/0.012 ms
```

Surprisingly, yes it does. It can even ping 1.1.1.1. I recall that in Linux, net.ipv4.ip_forward has to be explicitly enabled for traffic to pass between interfaces. A quick Google search, and I learn that FRR has an equivalent:

```
router3.as102# show ip forward
IP forwarding is off
router3.as102# conf t
router3.as102(config)# ip forwarding
```

Now to retry that ping:

```console
root@eyeball:/# ping 1.1.1.1 -c 2
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=59 time=0.137 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=59 time=0.076 ms

--- 1.1.1.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 0.076/0.106/0.137/0.030 ms
```

Solved!

[Back to Task Index](#task-index)

---

### Task 3

> HTTP requests from server3.dc1 to the DC-level anycast address 111.11.2.2 are consistently routed to the same server, causing overload. Investigate the cause of this behavior and resolve the issue.

I'll try sending some HTTP requests from server3.dc1 to 111.11.2.2.

```console
root@server3:/# for i in {1..10} ; do curl --local-port 50000-60000 -s http://111.11.2.2/ ; done;
server2.dc1
server1.dc1
server2.dc1
server1.dc1
server1.dc1
server2.dc1
server1.dc1
server1.dc1
server2.dc1
server1.dc1
```

This seems to be working as intended; traffic is being multipathed to the other two servers. And multipath is configured on the upstream routers:

```
root@router2:/# ip route show match 111.11.2.2
default via 172.20.20.1 dev eth0
111.11.2.0/24 proto bird metric 32
	nexthop via 10.1.1.2 dev eth1 weight 1
	nexthop via 10.1.3.2 dev eth2 weight 1
...
root@router3:/# ip route show match 111.11.2.2
default via 172.20.20.1 dev eth0
111.11.2.0/24 proto bird metric 32
	nexthop via 10.1.2.2 dev eth1 weight 1
	nexthop via 10.1.4.2 dev eth2 weight 1
```

So the only thing I can think of that might be causing this hypothetical problem is that maybe all the hypothetical HTTP requests are originating from the same source port? Usually, multipath uses a hashing algorithm on source/destination IPs and ports in order to keep any single connection using the same path stably--this would be a feature, not a bug. Since the task is hypothetical and there aren't any real applications that are overloading "the same server" that can be tweaked, I'm not sure what else to investigate.

Ticket closed I guess?

[Back to Task Index](#task-index)

---

### Task 4

> Identify and resolve the issue preventing server2.dc2 from handling any anycast requests.

#### Process & Resolution

First, I want to check the BGP state on server2.dc2.

```console
root@server2:/# birdc show protocols
BIRD 2.0.12 ready.
Name       Proto      Table      State  Since         Info
device1    Device     ---        up     16:26:06.861
direct1    Direct     ---        up     16:26:06.861
kernelv4   Kernel     master4    up     16:26:06.861
kernelv6   Kernel     master6    up     16:26:06.861
router2    BGP        ---        start  16:26:06.861  Connect
router3    BGP        ---        start  16:26:06.861  Connect
```

It's stuck in Connect. The TCP connection is not getting established at all. I checked the configuration on either side of the link to make sure IPs are correct (they are) and that the other side is reachable with ping (it is). Then I checked `ss`:

```
root@server2:/# ss -tpn
State      Recv-Q    Send-Q       Local Address:Port        Peer Address:Port   Process
SYN-SENT   0         1            10.2.4.2%eth2:50817           10.2.4.1:179     users:(("bird",pid=43,fd=8))
SYN-SENT   0         1            10.2.3.2%eth1:59559           10.2.3.1:179     users:(("bird",pid=43,fd=9))
root@server2:/# ss -tln
State   Recv-Q   Send-Q     Local Address:Port      Peer Address:Port  Process
LISTEN  0        511              0.0.0.0:80             0.0.0.0:*
LISTEN  0        8                0.0.0.0:179            0.0.0.0:*
LISTEN  0        4096          127.0.0.11:44517          0.0.0.0:*
LISTEN  0        511                 [::]:80                [::]:*
```

BIRD is listening on 179, and SYNs are being sent to 179. The output on the other side of the link looks the same. Something else must be blocking that port. `iptables`?

```console
root@server2:/# iptables -S
-P INPUT ACCEPT
-P FORWARD ACCEPT
-P OUTPUT ACCEPT
-A INPUT -p tcp -m tcp --dport 179 -j DROP
-A OUTPUT -p tcp -m tcp --dport 179 -j DROP
```

That's the issue: TCP is getting blocked on both the INPUT and OUTPUT chains specifically on port 179. Who would do such a thing??

```console
root@server2:/# iptables --flush INPUT
root@server2:/# iptables --flush OUTPUT
root@server2:/# iptables -S
-P INPUT ACCEPT
-P FORWARD ACCEPT
-P OUTPUT ACCEPT
root@server2:/# birdc show protocols
BIRD 2.0.12 ready.
Name       Proto      Table      State  Since         Info
device1    Device     ---        up     16:26:06.861
direct1    Direct     ---        up     16:26:06.861
kernelv4   Kernel     master4    up     16:26:06.861
kernelv6   Kernel     master6    up     16:26:06.861
router2    BGP        ---        up     02:28:45.769  Established
router3    BGP        ---        up     02:28:58.176  Established
```

Task complete!

[Back to Task Index](#task-index)

---

### Task 5

> server2.dc2 is not receiving any anycast traffic for IPv6 address fc00:aa::. Investigate the cause and fix the issue to restore proper traffic flow.

#### Process & Resolution

My first thought is to see what routes the routers have for that address.

```console
root@router3:/# ip -6 r show match fc00:aa::
fc00:aa::/48 proto bird metric 32 pref medium
	nexthop via fc00:dc2::2:2 dev eth1 weight 1
	nexthop via fc00:dc2::6:2 dev eth3 weight 1
fc00:aa::/46 via fc00:dc2::4:2 dev eth2 proto bird metric 32 pref medium
```

Looks pretty straightforward--server2.dc2 is advertising a route with a lower prefix length than the other two servers. I'll verify addressing on server2's loopback interface, and update it to use a /48 as well if needed.

```console
root@server2:/# ip -6 -br a
lo               UNKNOWN        fc00:aa::/46 fc00:22:3::/64 fc00:22:2::/64 fc00:dc2::255:2/128 ::1/128
eth0@if539       UP             3fff:172:20:20::3/64 fe80::a0b5:17ff:fe9b:693f/64
eth1@if548       UP             fc00:dc2::3:2/126 fe80::a8c1:abff:fea3:319/64
eth2@if552       UP             fc00:dc2::4:2/126 fe80::a8c1:abff:fe45:ed1d/64
root@server2:/# ip -6 a del fc00:aa::/46 dev lo
root@server2:/# ip -6 a add fc00:aa::/48 dev lo
```

The routes should look better now on the router:

```console
root@router3:/# ip -6 r show match fc00:aa::
fc00:aa::/48 proto bird metric 32 pref medium
	nexthop via fc00:dc2::2:2 dev eth1 weight 1
	nexthop via fc00:dc2::4:2 dev eth2 weight 1
	nexthop via fc00:dc2::6:2 dev eth3 weight 1
```

[Back to Task Index](#task-index)

---

### Task 6

> server3.dc2 cannot access the DC-level anycast service at fc00:22:2::. Identify the cause of the problem and implement a solution to restore connectivity.

#### Process

First, I want to see the problem for myself. I'll check the routing table on both server2 and server 3 to see what's going on.

_Using `fc00::/26` shows only routes to networks between `fc00:0000::` and `fc00:003f::`, effectively filtering out all the noise of routes to infrastracture and global anycast addresses._

```console
root@server2:/# ip -6 r show to root fc00::/26
fc00:11:1::/64 proto bird metric 32 pref medium
	nexthop via fc00:dc2::3:1 dev eth1 weight 1
	nexthop via fc00:dc2::4:1 dev eth2 weight 1
fc00:11:2::/64 proto bird metric 32 pref medium
	nexthop via fc00:dc2::3:1 dev eth1 weight 1
	nexthop via fc00:dc2::4:1 dev eth2 weight 1
fc00:11:3::/64 proto bird metric 32 pref medium
	nexthop via fc00:dc2::3:1 dev eth1 weight 1
	nexthop via fc00:dc2::4:1 dev eth2 weight 1
fc00:22:1::/64 proto bird metric 32 pref medium
	nexthop via fc00:dc2::3:1 dev eth1 weight 1
	nexthop via fc00:dc2::4:1 dev eth2 weight 1
fc00:22:2::/64 dev lo proto bird metric 32 pref medium
fc00:22:2::/64 dev lo proto kernel metric 256 pref medium
fc00:22:3::/64 dev lo proto bird metric 32 pref medium
fc00:22:3::/64 dev lo proto kernel metric 256 pref medium

root@server3:/# ip -6 r show to root fc00::/26
fc00:11:1::/64 proto bird metric 32 pref medium
	nexthop via fc00:dc2::5:1 dev eth1 weight 1
	nexthop via fc00:dc2::6:1 dev eth2 weight 1
fc00:11:2::/64 proto bird metric 32 pref medium
	nexthop via fc00:dc2::5:1 dev eth1 weight 1
	nexthop via fc00:dc2::6:1 dev eth2 weight 1
fc00:11:3::/64 proto bird metric 32 pref medium
	nexthop via fc00:dc2::5:1 dev eth1 weight 1
	nexthop via fc00:dc2::6:1 dev eth2 weight 1
fc00:22:1::/64 dev lo proto bird metric 32 pref medium
fc00:22:1::/64 dev lo proto kernel metric 256 pref medium
fc00:22:3::/64 dev lo proto bird metric 32 pref medium
fc00:22:3::/64 dev lo proto kernel metric 256 pref medium
```

I see that server2 is getting a route to `fc00:22:1::/64` from BIRD, but router3 is not analogously receiving a route to `fc00:22:2::/64`. I strongly suspect the connected routers are advertising it though:

```console
root@router2:/# birdc show route for fc00:22:2:: export router3
BIRD 2.0.12 ready.
Table master6:
fc00:22:2::/64       unicast [server1 17:01:06.484 from 10.2.1.2] * (100) [AS65201i]
	via fc00:dc2::1:2 on eth1
```

So a connected router is advertising a route to `fc00:22:2::/64` to server3, but BIRD isn't adding it to the routing table. To be honest, at this point, I had to compare `bird.conf` on server2 vs server3 to see what was up. Turns out, the peer configuration for the routers on server3 was missing one line:

```
# bird.conf (partial)

protocol bgp router3 {
        local 10.2.4.2 as 65201;
        neighbor 10.2.4.1 as 65200;
        allow local as;               # this line is missing on router3.dc2
        ipv4 {
                export all;
                import all;
        };
        ipv6 {
                export all;
                import all;
        };
}
```

This makes sense! Normally, when a router receives a route advertisement from an eBGP peer, and that advertisement's AS_PATH contains the router's own ASN, it discards the route. Normally this is essential for loop prevention--when a router sees its own ASN in a path, it knows that that advertisement has been here before, it has come full-circle--there's usually no need to recirculate that advertisement (congesting BGP) with long, useless, convoluted paths.

This BGP Anycast scenario is different, though. The AS containing the servers is not actually an "autonomous" network--the servers aren't even connected to one another within the AS. The only way for traffic to get from one server to another is for it to leave and re-enter the AS. It is safe, too: the servers are leaves in the topology, and there's no risk of them reemitting those advertisements and creating a loop.

So I will add the missing lines and verify that server3 now has a route to `fc00:22:2::`.

```console
root@server3:/# vim etc/bird/bird.conf
root@server3:/# birdc configure check
BIRD 2.0.12 ready.
Reading configuration from /etc/bird/bird.conf
Configuration OK
root@server3:/# birdc configure
BIRD 2.0.12 ready.
Reading configuration from /etc/bird/bird.conf
Reconfiguration in progress
root@server3:/# ip -6 r show to root fc00::/26
fc00:11:1::/64 proto bird metric 32 pref medium
	nexthop via fc00:dc2::5:1 dev eth1 weight 1
	nexthop via fc00:dc2::6:1 dev eth2 weight 1
fc00:11:2::/64 proto bird metric 32 pref medium
	nexthop via fc00:dc2::5:1 dev eth1 weight 1
	nexthop via fc00:dc2::6:1 dev eth2 weight 1
fc00:11:3::/64 proto bird metric 32 pref medium
	nexthop via fc00:dc2::5:1 dev eth1 weight 1
	nexthop via fc00:dc2::6:1 dev eth2 weight 1
fc00:22:1::/64 dev lo proto bird metric 32 pref medium
fc00:22:1::/64 dev lo proto kernel metric 256 pref medium
fc00:22:2::/64 proto bird metric 32 pref medium
	nexthop via fc00:dc2::5:1 dev eth1 weight 1
	nexthop via fc00:dc2::6:1 dev eth2 weight 1
fc00:22:3::/64 dev lo proto bird metric 32 pref medium
fc00:22:3::/64 dev lo proto kernel metric 256 pref medium
```

I will remember this pattern for next time!

[Back to Task Index](#task-index)

---

### Task 7

> Customer requests from AS 101 to the anycast address 2.2.2.2 are always routed to Data Center #2 through AS 102, even when Data Center #1 is closer. Investigate the routing behavior and implement a solution to ensure traffic is directed to the nearest data center.

#### Process & Resolution

I will enter the only router in AS101 and have a look at path selection for 2.2.2.2:

```console
router1.as101# show bgp ipv4 2.2.2.2
BGP routing table entry for 2.2.2.0/24, version 27
Paths: (2 available, best #1, table default)
  Advertised to peers:
  10.101.1.2 10.102.2.1
  102 103 1443 65200 65201
    10.102.2.1 from 10.102.2.1 (10.102.254.1)
      Origin IGP, valid, external, best (AS Path)
      Last update: Tue Sep 29 16:26:17 2026
  1443 1443 1443 1443 65100 65101
    10.101.1.2 from 10.101.1.2 (10.1.254.1)
      Origin IGP, valid, external
      Last update: Tue Sep 29 16:26:17 2026
```

Requests to 2.2.2.2 are getting routed through AS102 because DC1 is prepending heavily--DC1 seems to be trying to steer requests to 2.2.2.2 away from itself.

It would be prettier to delete that prepending, and I have the agency to do that; but I feel like the wording of the task is insinuating that the issue is with customers of AS101 and that I should do my work locally within the AS. So I will act as an engineer in AS101 and steer traffic to DC1 with LOCAL_PREF.

```
router1.as101# conf t
router1.as101(config)# ip prefix-list STEERED_SUBNETS seq 10 permit 2.2.2.0/24
router1.as101(config)# route-map AS1443_IN permit 10
router1.as101(config-route-map)# match ip address prefix-list STEERED_SUBNETS
router1.as101(config-route-map)# set local-preference 110
router1.as101(config-route-map)# exit
router1.as101(config)# router bgp 101
router1.as101(config-router)# address-family ipv4
router1.as101(config-router-af)# neighbor 10.101.1.2 route-map AS1443_IN in
router1.as101(config-router-af)# end
router1.as101# show bgp ipv4 2.2.2.2
BGP routing table entry for 2.2.2.0/24, version 56
Paths: (2 available, best #1, table default)
  Advertised to peers:
  10.101.1.2 10.102.2.1
  1443 1443 1443 1443 65100 65101
    10.101.1.2 from 10.101.1.2 (10.1.254.1)
      Origin IGP, localpref 110, valid, external, best (Local Pref)
      Last update: Wed Sep 30 05:19:00 2026
  102 103 1443 65200 65201
    10.102.2.1 from 10.102.2.1 (10.102.254.1)
      Origin IGP, valid, external
      Last update: Tue Sep 29 16:26:17 2026
```

Now requests to 2.2.2.2 will go to the closest datacenter. Sorry DC1.

[Back to Task Index](#task-index)

---

### Task 8

> Customer requests from AS 103 to the anycast address 1.1.1.1 are consistently routed to Data Center #1 via AS 102, despite the proximity of Data Center #2. Investigate the routing behavior and implement a solution to ensure traffic is directed to the nearest data center.

First, let's see how router1.as103 is selecting a path for 1.1.1.1.

```
router1.as103# show bgp ipv4 1.1.1.1
BGP routing table entry for 1.1.1.0/24, version 67
Paths: (1 available, best #1, table default)
  Advertised to peers:
  10.102.7.1 10.103.1.2
  102 1443 1443 1443 65200 65201
    10.102.7.1 from 10.102.7.1 (10.102.254.2)
      Origin IGP, valid, external, best (First path received)
      Last update: Wed Sep 30 05:19:00 2026
```

Strange that there is only one path available to 1.1.1.1! It looks like it was actually originated from DC2, but it's coming from AS102. Am I not getting any routes from DC2?

```
router1.as103# show bgp ipv4 neighbors 10.103.1.2 received-routes
BGP table version is 85, local router ID is 10.103.254.1, vrf id 0
Default local pref 100, local AS 103
Status codes:  s suppressed, d damped, h history, u unsorted, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *> 1.1.1.0/24       10.103.1.2                             0 1443 65200 65201 i
 *> 2.2.2.0/24       10.103.1.2                             0 1443 65200 65201 i
 *> 10.2.1.0/30      10.103.1.2                             0 1443 65200 i
 *> 10.2.2.0/30      10.103.1.2                             0 1443 65200 i
 *> 10.2.3.0/30      10.103.1.2                             0 1443 65200 i
 *> 10.2.4.0/30      10.103.1.2                             0 1443 65200 i
 *> 10.2.5.0/30      10.103.1.2                             0 1443 65200 i
 *> 10.2.6.0/30      10.103.1.2                             0 1443 65200 i
 *> 10.2.7.0/30      10.103.1.2                             0 1443 65200 i
 *> 10.2.8.0/30      10.103.1.2               0             0 1443 i
 *> 10.2.9.0/30      10.103.1.2               0             0 1443 i
 *> 10.2.254.1/32    10.103.1.2               0             0 1443 i
 *> 10.2.254.2/32    10.103.1.2                             0 1443 65200 i
 *> 10.2.254.3/32    10.103.1.2                             0 1443 65200 i
 *> 10.2.255.1/32    10.103.1.2                             0 1443 65200 65201 i
 *> 10.2.255.2/32    10.103.1.2                             0 1443 65200 65201 i
 *> 10.2.255.3/32    10.103.1.2                             0 1443 65200 65201 i
 *> 222.22.1.0/24    10.103.1.2                             0 1443 65200 65201 i
 *> 222.22.2.0/24    10.103.1.2                             0 1443 65200 65201 i
 *> 222.22.3.0/24    10.103.1.2                             0 1443 65200 65201 i

Total number of prefixes 20 (1 filtered)
```

In fact I'm getting plenty of routes from DC2, including one for 1.1.1.0/24. But what's up with that `(1 filtered)` at the end? Is it a bad route-map?

```
router1.as103# show run
...
!
ip prefix-list DENY_LIST seq 10 deny 1.1.1.0/24
ip prefix-list DENY_LIST seq 20 permit 0.0.0.0/0 le 32
!
route-map ALLOW_ALL permit 10
exit
!
route-map AS1334_IN permit 10
 match ip address prefix-list DENY_LIST
exit
!
router bgp 103
 bgp router-id 10.103.254.1
 neighbor 10.102.7.1 remote-as 102
 neighbor 10.103.1.2 remote-as 1443
 !
 address-family ipv4 unicast
  network 10.103.1.0/30
  network 10.103.254.1/32
  network 192.168.103.0/24
  aggregate-address 10.103.0.0/16 summary-only
  neighbor 10.102.7.1 soft-reconfiguration inbound
  neighbor 10.102.7.1 route-map ALLOW_ALL in
  neighbor 10.102.7.1 route-map ALLOW_ALL out
  neighbor 10.103.1.2 soft-reconfiguration inbound
  neighbor 10.103.1.2 route-map AS1334_IN in
  neighbor 10.103.1.2 route-map ALLOW_ALL out
 exit-address-family
```

Of course 1.1.1.1 isn't being routed to DC2. For some reason it has been configured to explicitly filter out routes to 1.1.1.0/24 received from DC2. Let's just allow all ipv4 routes from DC2.

```
router1.as103# conf t
router1.as103(config)# router bgp 103
router1.as103(config-router)# address-family ipv4 unicast
router1.as103(config-router-af)# neighbor 10.103.1.2 route-map ALLOW_ALL in
router1.as103(config-router-af)# end
router1.as103# show bgp ipv4 1.1.1.1
BGP routing table entry for 1.1.1.0/24, version 86
Paths: (1 available, best #1, table default)
  Advertised to peers:
  10.102.7.1 10.103.1.2
  1443 65200 65201
    10.103.1.2 from 10.103.1.2 (10.2.254.1)
      Origin IGP, valid, external, best (First path received)
      Last update: Wed Sep 30 05:44:34 2026
```

All tasks completed!

[Back to Task Index](#task-index)
