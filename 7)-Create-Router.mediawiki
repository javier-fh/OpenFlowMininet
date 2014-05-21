In this exercise, you'll make a static layer-3 forwarder/switch.  It's not exactly a router, because it won't decrement the IP TTL and recompute the checksum at each hop (so traceroute won't work).  Actions to do TTL and checksum modification are expected in the upcoming OpenFlow version 1.1.  However, it will match on masked IP prefix ranges, just like a real router.

From [http://www.faqs.org/rfcs/rfc1812.html#ixzz0oPra4P9C RFC 1812]:
<blockquote>
An IP router can be distinguished from other sorts of packet switching devices in that a router examines the IP protocol header as part of the switching process. It generally removes the Link Layer header a message was received with, modifies the IP header, and replaces the Link Layer header for retransmission.
</blockquote>

To simplify the exercise, your "router" will be completely static. With no BGP or OSPF, you'll have no need to send or receive route table updates.

Each network node will have a configured subnet.  If a packet is destined for a host within that subnet, the node acts as a switch and forwards the packet with no changes, to a known port or broadcast, just like in the previous exercise.  If a packet is destined for some IP address for which the router knows the next hop, it should modify the layer-2 destination and forward the packet to the correct port.

Our hope is that this exercise will show that with an OpenFlow-enabled forwarding device, the network is effectively layerless; you can mix switch, router, and higher-layer functionality.

Please note that this is an advanced exercise, and given that most implementation details are up to you, will be harder.

== Create Topology ==

You'll need a slightly different topology, something like this:
'''Note: For Mininet 2.0, the hosts have been renumbered h1-h3 and 10.1-10.3.'''

[[File:Router_topo.png]]


… which will need to be described in a way that Mininet will understand.

There's an example custom topology at:
 ~/mininet/custom/topo-2sw-2host.py

First, copy the example to a new file:
 $ cp ~/mininet/custom/topo-2sw-2host.py mytopo.py

To run a custom topology, pass Mininet the custom file and pass in the custom topology:
 $ sudo mn --custom mytopo.py --topo mytopo --mac

Then, in the Mininet console, run:
 mininet> pingall

Now, modify your topology file to match the picture and verify full host connectivity with pingall in the Mininet console.

== Set up hosts ==

<!--

First, you may need to update to the latest version of Mininet (which supports CLI scripting and the <tt>--prefixlen</tt> option, which you may find to be useful.)

 $ pushd ~/mininet; git stash; git checkout master; git pull; sudo make install; popd

-->

Set up IP configuration on each virtual host to force each one to send to the gateway for destination IPs that are outside of their configured subnet.

You'll need to configure each '''host''' with a subnet, IP, gateway, and netmask. 

''It may seem obvious, but we will warn you anyway: do not attempt to assign IP addresses to the interfaces belonging to switches s1 and s2. If you need to handle traffic "to" or "from" a switch, do so using OpenFlow.''

There are several ways that this can be done:

* '''Unix commands via Mininet CLI'''

From within the Mininet CLI, you can send commands to hosts to configure them:

 mininet> h1 ifconfig h2-eth0 10.0.1.1/24
 mininet> h1 route add default gw 10.0.1.1
 mininet> h1 route -n

* '''Regular Unix commands via an xterm'''

Another way of configuring hosts is to open up an xterm

 mininet> xterm h1

and then to type into that xterm window to configure the host:

 # ifconfig h1-eth0 10.0.1.1/24

* '''Running a configuration script'''

You can also run scripts on the hosts themselves, for example, if you have a script called config_script, you can run it on host h2 from the Mininet CLI:

 mininet> h1 config_script

If you write such a script, you can either have it automatically know what to do (e.g. based on MAC address, which you can assign based on node number with --mac ) or pass in additional parameters as desired.

* '''CLI scripting'''

To avoid repetitive CLI typing, it is possible to create a file consisting of multiple CLI commands, and then run that script in Mininet:

 mininet> source my_cli_script
or
 # mn --pre my_cli_script

A sample CLI script might look like:

 py "Configuring network"
 h1 ifconfig h3-eth0 10.0.1.1/24
 h2 ifconfig h4-eth0 10.0.1.2/24
 h3 ifconfig h5-eth0 10.0.2.3/24
 h1 route add default gw 10.0.1.1
 h2 route add default gw 10.0.1.1
 h3 route add default gw 10.0.2.1
 py "Current network:"
 net
 dump

CLI scripting is designed to save typing rather than to be a full programming environment (although you can evaluate Python expressions from the CLI, as shown above using the 'py' command!) For advanced tasks, the Mininet API provides flexible access to all Mininet capabilities.

* '''Mininet API'''
Mininet's Python API provides a much more powerful means of creating and configuring networks programatically, but is probably beyond the scope of this tutorial. Some examples of using the Mininet API may be found in <tt>mininet/examples</tt>, and they are described in <tt>mininet/examples/README</tt>.

* '''Changing the node IDs in the topology'''

It is also possible to change the default IP addresses by changing node IDs in a topology.

The node ID in a topology is the default IP address in base 256. 
For example, setting node ID 258 will result in a default IP address of 10.0.1.2.

The default IP prefix length may be changed with the command-line option --prefixlen. For example,

 # mn --prefixlen 24

will use /24 as the default prefix length (i.e. netmask of 255.255.255.0).

For the this tutorial, you may find that using CLI scripting or changing the node IDs (and setting the prefix length to 24) may be most convenient.

Once you have your new topology configured and connected to your pytutorial controller, you should be able to ping between hosts on the same subnet, but not between hosts on different subnets. Your job is to create a new (or revised) controller that installs rules to forward packets between the two subnets!

== ARP ==

A router generally has to respond to ARP requests. You will see ethernet broadcasts which will (initially at least) be forwarded to the controller.

Your controller should probably construct ARP replies and forward them out the appropriate ports.

It may be possible to avoid dealing with ARP (at least initially) by adding static ARP entries in each host for all of the nodes on its subnet.

== Static Routing ==

Once ARP has been handled, you will need to handle routing for the static configuration. Since we know what is connected to each port, we can match on IP address (or prefix, in the case of the remote subnet) and forward out the appropriate port.

In the case of the left switch, for example, we will want to forward packets destined for an attached host to the appropriate port, while packets destined for the remote subnet should be forwarded to the right switch.

== ICMP ==

Additionally, your controller may receive ICMP echo (ping) requests for each switch, which it should respond to.

Lastly, packets for unreachable subnets should be responded to with ICMP network unreachable messages.

== Testing your router ==

If the router works properly:
* attempts to send from 10.0.1.2 to an unknown address range like 10.99.0.1 should yield an [http://en.wikipedia.org/wiki/ICMP_Destination_Unreachable ICMP destination unreachable] message.
* packets sent to hosts on the same subnet should be treated like before.
* packets sent to hosts on a known address range should have their MAC dst field changed to that of the next-hop router.
* the router should be pingable, and should generate an ICMP echo reply in response to an ICMP echo request.

The exercise is meant to give more practice with packet parsing and show how to use OpenFlow to modify packets.

Let the instructor know when you get stuck or when you complete the exercise.  This one is more open, and you'll need different actions (MAC dst modify) and matching (IP dst/prefix combinations), for which NOX core.py should help.
