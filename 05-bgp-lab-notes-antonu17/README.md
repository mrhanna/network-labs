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

## Tasks

**Jump to Task**: [Task 1](#task-1) - [Task 2](#task-2) - [Task 3](#task-3) - [Task 4](#task-4) - [Task 5](#task-5) - [Task 6](#task-6) - [Task 7](#task-7)

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

[Back to Tasks](#tasks)

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

[Back to Tasks](#tasks)

---

### Task 3

> Currently, requests from AS 102 customers are distributed between two Data Centers. Configure routing so that Data Center #2 handles all requests to the network 2.2.2.0/24 from AS 102. Ensure that requests from AS 101 customers to the same network continue to be partially served by Data Center #1. Note: Modifications to transit provider configurations are not allowed.

#### Process

Without access to transit provider configuration, and since the lab transit providers doen't have communities set up that allow me to request a local_pref, my best bet is probably just to do some prepending to the AS_paths. (MED almost could be an option here since both DCs share the same ASN, but since MED is non-transitive, it wouldn't be able to pass through all the transit providers. Besides [all the other caveats with MED](https://ine.com/blog/2011-10-12-understanding-bgp-med-and-bgp-deterministic-med)).

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

[Back to Tasks](#tasks)

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

[Back to Tasks](#tasks)

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

[Back to Tasks](#tasks)

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

[Back to Tasks](#tasks)

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

[Back to Tasks](#tasks)
