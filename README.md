# Description

This lab replicates the original SIGCOMM2015 one but in Mininet. This is a small
simple topology (with only one link with an high cost):

```
                         +----+
                         | D1 |
                         +----+
                          |
            +---+        +---+      +---+
            | R2|--------|R3 |------|D2 |
            +---+        +---+      +---+
              |            |
           10 |            |
              |            |
 +--+      +----+        +---+        +--+
 |S1|------| R1 |--------| R4|--------|C1|
 +--+      +----+        +---+        +--+
              |
            +---+
            |S2 |
            +---+
```

# Example (to produce the SIGCOMM2015 graphs)

Open a terminal in this directory.

```bash
python network.py
```

Wait until the network has converged (an easy indicator is to look for the logs
looking like: [   INFO] update_graph: LSA update yielded +1 -0 edges changes,
once these stop appearing, the OSPF topology is stable).

```bash
mininetcli: xterm s1 s2 d1 d2
d1: iperf -u -s -i .5
d2: iperf -u -s -i .5
mininetcli: d1 ip a
s1: iperf -u -b 1M -t 300 -c <ip of d1-eth0 from previous command>
```

You should now see in the xterm of d1 that it is receiving packet at rate close
to 1Mb/s.

```bash
mininetcli: d2 ip a
s2: iperf -u -b 1M -t 300 -c <ip of d2-eth0 from previous command>
```

You should now see that both d1 and d2 are receiving traffic, however the
reported rates are instead close to 500kb/s. Indeed, r3 is the penultimate hop
for both destination. As a result, the SPT computed by the routers is such that
the paths between r1 and r3 are the same: r1-r4-r3 (due to the high cost
between r1 and r2). As we artificially limited the bandwidth of the links to
1Mb/s, we have created congestion. You can verify this by using traceroute,
which should report twice the same  path. The routing tables of the routers
can also be observed for that purpose.

```bash
mininetcli: r1 traceroute -4n d1
mininetcli: r1 traceroute -4n d2
mininetcli: route d1
mininetcli: route d2
```

Open a new terminal window to the directory of this lab.

```bash
python network -c
```

This will instruct the Fibbing controller to enforce 2 distinct paths in the
network:
    1. r1-r2-r3-d1
    2. r1-r4-r3-d2
Since these paths are link-disjoint, the congestion should be removed, which
should be visible in the xterms of d1 and d2. Again, you can verify this by
looking at the routers routing tables and/or traceroute from the mininet cli
as done previously.

# Usage

First, create the network by running:

```bash
python network.py -n
```

The first command will spawn the network (square topology), and you will land
in the mininet CLI. The second on will spawn an instance of a controller running
the Simple algorithm.

Additionally, you can append a --debug parameter to received detailed logs.

Once you see that the network has been created, and that the southbound
Fibbing controller seems to have a stable view of the topology (the log entries
'[   INFO] update_graph: LSA update yielded +1 -0 edges changes' should stop
when that is the case), you can start the 'application' component of the
controller:

```bash
python network.py -c
```

In this case, it simply spreads the traffic going towards d1 and d2.

# Misc.

Beside the 'usual' mininet cli command, 2 extra ones can be useful:
    1. route <destination> : Will output the routing table towards the given
    destination for all routers.
    Example: route d1

    2. ip <address> : Will show the node name for the given ip address

Additional commands will be listed in the 'help' command.

The quickest way to test if the controller is working properly is to run
traceroute before and after spawning the 'application' controller
(network.py -c).

```bash
s1 traceroute -4n d1
s1 traceroute -4n d2
```

The routes should no longer be identical to reach d1 and d2 once the application
controller is started.

If you want to obtain the 'graphs' that can be seen in the paper or in the
demo on the website, you can run UDP iperf sessions between s1-d1 and s2-d2 and
monitor the achieved throughput on the server logs, or watch it in realtime
using e.g. 'speedometer d1-eth0' in another console.
(after having spawned a xterm for d1 via xterm d1)
