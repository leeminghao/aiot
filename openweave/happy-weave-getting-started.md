## Get started

Install toolchain and dependencies
To build Happy and Weave, make sure you have a supported toolchain and all dependencies installed.

$ sudo apt-get update
$ sudo apt-get install -y autotools-dev build-essential git lcov \
                           libdbus-1-dev libglib2.0-dev libssl-dev \
                           libudev-dev make python2.7 software-properties-common \
                           python-setuptools bridge-utils python-lockfile \
                           python-psutil
$ sudo apt-get install -y --force-yes gcc-arm-none-eabi
$ sudo apt-get update -qq
Get the source code
Clone the Happy and OpenWeave Git repositories from the command line:

$ cd ~
$ git clone https://github.com/openweave/happy.git
$ git clone https://github.com/openweave/openweave-core.git
Install Happy
From the Happy root directory, install Happy:

$ cd ~/happy
$ make
Verify Happy installation
Happy commands should now be accessible from the command line:

$ happy-state
State Name:  happy

NETWORKS   Name         Type   State                                     Prefixes

NODES      Name    Interface    Type                                          IPs
Install OpenWeave
From the OpenWeave root directory, install OpenWeave:

$ cd ~/openweave-core
$ make -f Makefile-Standalone
Configure Happy with OpenWeave
To use OpenWeave with Happy, you need to let Happy know where to find the Weave installation. Update the Happy configuration with the path to /src/test-apps within your OpenWeave build:

$ happy-configuration weave_path ~/openweave-core/build/x86_64-unknown-linux-gnu/src/test-apps
Note: Your build path may be different than what is shown here (x86_64-unknown-linux-gnu). Double-check the target folder in ~/openweave-core/build/ and ensure the weave_path configuration is correct for your Linux machine.

Confirm the configuration:

$ happy-configuration
User Happy Configuration
        weave_path         ~/openweave-core/build/x86_64-unknown-linux-gnu/src/test-apps
Verify OpenWeave installation
Weave commands needed in this Codelab should be accessible from the command line:

$ weave-fabric-add -h

    weave-fabric-add creates a weave fabric.

    weave-fabric-add [-h --help] [-q --quiet] [-i --id <FABRIC_ID>]

    Example:
    $ weave-fabric-add 123456
        Creates a Weave Fabric with id 123456

    return:
        0    success
        1    fail
If you get the error weave-fabric-add: command not found, update your PATH environment variable with the path used for Happy binaries:

$ export PATH=$PATH:~/openweave-core/src/test-apps/happy/bin

## Your first topology

Let's create the following three-node topology with Happy.

This topology is an example of a simple Home Area Network (HAN). In this HAN, two nodes are connected together in a Thread network, and one of those nodes connects to a third via Wi-Fi. This node can also be connected to a wireless router in the home to provide internet connectivity for the entire HAN. More about this later.

First, create the three nodes:

$ happy-node-add 01ThreadNode
$ happy-node-add 02BorderRouter
$ happy-node-add 03WiFiNode
Tip: Use the -h flag with any Happy command for help.

Let's make sure they exist:

$ happy-node-list
01ThreadNode
02BorderRouter
03WiFiNode
Note: Each line of output may be prefixed with a timestamp. In example output throughout this Codelab, the timestamp is omitted.

Now let's create some networks:

$ happy-network-add ThreadNetwork thread
$ happy-network-add WiFiNetwork wifi
Verify that the networks exist:

$ happy-network-list
ThreadNetwork
WiFiNetwork
Check the Happy state:

$ happy-state

State Name:  happy

NETWORKS   Name         Type   State                                     Prefixes
  ThreadNetwork       thread      UP
    WiFiNetwork         wifi      UP

NODES      Name    Interface    Type                                          IPs
   01ThreadNode
 02BorderRouter
     03WiFiNode
It's not enough just to bring a network up—we have to add nodes to the networks. Following our topology diagram, add each node to the appropriate network(s):

$ happy-node-join 01ThreadNode ThreadNetwork
$ happy-node-join 02BorderRouter ThreadNetwork
$ happy-node-join 02BorderRouter WiFiNetwork
$ happy-node-join 03WiFiNode WiFiNetwork
Note that 02BorderRouter was added to both ThreadNetwork and WiFiNetwork. That's because as a Border Router within our HAN, this node connects the two individual networks together.

Check the Happy state. Each node's interfaces are up:

$ happy-state

State Name:  happy

NETWORKS   Name         Type   State                                     Prefixes
  ThreadNetwork       thread      UP
    WiFiNetwork         wifi      UP

NODES      Name    Interface    Type                                          IPs
   01ThreadNode        wpan0  thread
 02BorderRouter        wpan0  thread
                       wlan0    wifi
     03WiFiNode        wlan0    wifi
Our topology now looks like this:



The last step in bringing up our Happy network is to assign IP addresses to each interface on each node. Specify the IP prefix for a network, and Happy automatically assigns IP addresses for you.

Because the Thread protocol uses IPv6, add an IPv6 prefix to the Thread network:

$ happy-network-address ThreadNetwork 2001:db8:1:2::
Note: The 2001:db8::/32 IPv6 prefix is reserved for documentation purposes, per RFC 3849. Individual IPv6 addresses on your system will differ from the examples shown here in this Codelab.

Check the Happy state. The Thread interfaces on each Thread node have IP addresses:

$ happy-state

State Name:  happy

NETWORKS   Name         Type   State                                     Prefixes
  ThreadNetwork       thread      UP                       2001:0db8:0001:0002/64

    WiFiNetwork         wifi      UP

NODES      Name    Interface    Type                                          IPs
   01ThreadNode        wpan0  thread   2001:0db8:0001:0002:3e36:13ff:fe33:732e/64

 02BorderRouter        wpan0  thread   2001:0db8:0001:0002:a651:3eff:fe92:6dbc/64

                       wlan0    wifi
     03WiFiNode        wlan0    wifi
For the WiFi network, add both IPv4 and IPv6 prefixes:

$ happy-network-address WiFiNetwork 2001:db8:a:b::
$ happy-network-address WiFiNetwork 10.0.1.0
Check the Happy state once more. All interfaces have assigned IP addresses, with two for each Wi-Fi interface:

$ happy-state

State Name:  happy

NETWORKS   Name         Type   State                                     Prefixes
  ThreadNetwork       thread      UP                       2001:0db8:0001:0002/64

    WiFiNetwork         wifi      UP                       2001:0db8:000a:000b/64
                                                                        10.0.1/24


NODES      Name    Interface    Type                                          IPs
   01ThreadNode        wpan0  thread   2001:0db8:0001:0002:3e36:13ff:fe33:732e/64

 02BorderRouter        wpan0  thread   2001:0db8:0001:0002:a651:3eff:fe92:6dbc/64

                       wlan0    wifi                                  10.0.1.2/24
                                       2001:0db8:000a:000b:426c:38ff:fe90:01e6/64

     03WiFiNode        wlan0    wifi   2001:0db8:000a:000b:9aae:2bff:fe71:62fa/64
                                                                      10.0.1.3/24
Here is our updated topology:

## Test the connectivity

Now that our Happy network is up and running, let's test its connectivity by pinging the other nodes from 01ThreadNode:

$ happy-ping 01ThreadNode 02BorderRouter
[Ping] ping from 01ThreadNode to 02BorderRouter on address
    10.0.1.2 -> 100% packet loss
[Ping] ping from 01ThreadNode to 02BorderRouter on address
    2001:0db8:0001:0002:a651:3eff:fe92:6dbc -> 0% packet loss
[Ping] ping from 01ThreadNode to 02BorderRouter on address
    2001:0db8:000a:000b:426c:38ff:fe90:01e6 -> 100% packet loss

$ happy-ping 01ThreadNode 03WiFiNode
[Ping] ping from 01ThreadNode to 03WiFiNode on address
    2001:0db8:000a:000b:9aae:2bff:fe71:62fa -> 100% packet loss
[Ping] ping from 01ThreadNode to 03WiFiNode on address
    10.0.1.3 -> 100% packet loss
The happy-ping command tries to ping every IP address for every interface on the target node. We can ignore the IPv4 addresses because Thread only uses IPv6.

Note that only one IPv6 ping was successful: the one on 02BorderRouter's wpan0 interface, which is the only address 01ThreadNode can directly reach:



The other IPv6 addresses failed because forwarding has not been enabled between wpan0 and wlan0 on 02BorderRouter. Thus, 01ThreadNode has no idea 03WiFiNode exists, or how to reach it. Happy has brought up the simulated network, but has not enabled all routing and forwarding between nodes.

Add routes
To route IPv6 traffic across the HAN, add the proper routes to each node in each network, in both directions (so the ping knows how to return to the source node).

For each node, you'll need to know:

the nearest network gateway—in this case, 02BorderRouter for both
the target network—where to go after the gateway
For our three node network, that gives us the following:

from Source Network

to Target Network

via Gateway

ThreadNetwork

WiFiNetwork

02BorderRouter wlan0

2001:db8:1:2::/64 prefix

WiFiNetwork

ThreadNetwork

02BorderRouter wpan0

2001:db8:a:b::/64 prefix

This can be done individually for each node with happy-node-route, but it's easier to do it for all nodes in each network with happy-network-route.

$ happy-network-route -a -i ThreadNetwork -t default -v 02BorderRouter -p 2001:db8:1:2::/64
$ happy-network-route -a -i WiFiNetwork -t default -v 02BorderRouter -p 2001:db8:a:b::/64
For an explanation of the command-line flags, use happy-network-route -h.

Note: We recommend using node names and default for addresses in the happy-node-route or happy-network-route commands, as this accounts for differing IP addresses between systems when saving or loading a topology. Hard-coded IP addresses in saved Happy topologies or scripts may not be available for use on every Linux system.

The happy-network-route command also turns on IPv4 and IPv6 forwarding for each node, as needed. This allows traffic to route from one interface to another within a node.

Now retry the ping:

$ happy-ping 01ThreadNode 02BorderRouter
[Ping] ping from 01ThreadNode to 02BorderRouter on address
    10.0.1.2 -> 100% packet loss
[Ping] ping from 01ThreadNode to 02BorderRouter on address
    2001:0db8:0001:0002:a651:3eff:fe92:6dbc -> 0% packet loss
[Ping] ping from 01ThreadNode to 02BorderRouter on address
    2001:0db8:000a:000b:426c:38ff:fe90:01e6 -> 0% packet loss
Both IPv6 pings work! With forwarding on, it knows how to reach the wlan0 interface. The IPv4 ping still fails, because we only configured IPv6 routes and forwarding (also because Thread doesn't run over IPv4).

Since we added network routes to both sides, let's ping across networks:

$ happy-ping 01ThreadNode 03WiFiNode
[Ping] ping from 01ThreadNode to 03WiFiNode on address
    2001:0db8:000a:000b:9aae:2bff:fe71:62fa -> 0% packet loss
[Ping] ping from 01ThreadNode to 03WiFiNode on address
    10.0.1.3 -> 100% packet loss
The IPv6 ping works as expected. You now have a fully-functional, simulated IPv6 HAN.

## Add Weave
Weave is a network application layer that provides the secure and reliable communications backbone for Nest products. We can add Weave functionality with OpenWeave, the open-source version of Weave.

An implementation of Weave is called a "fabric". A Weave fabric is a network that comprises all HAN nodes, the Nest Service, and any mobile devices participating in the HAN. It sits on top of the HAN and enables easier routing across the different underlying network link technologies (for example, Thread or Wi-Fi).

Create the Weave fabric for your HAN, using fab1 as the Fabric ID, then configure all nodes for Weave:

$ weave-fabric-add fab1
$ weave-node-configure
Now that Weave is configured, check the Happy state:

$ happy-state

State Name:  happy

NETWORKS   Name         Type   State                                     Prefixes
  ThreadNetwork       thread      UP                       2001:0db8:0001:0002/64

    WiFiNetwork         wifi      UP                       2001:0db8:000a:000b/64
                                                                        10.0.1/24


NODES      Name    Interface    Type                                          IPs
   01ThreadNode        wpan0  thread   2001:0db8:0001:0002:3e36:13ff:fe33:732e/64
                                       fd00:0000:fab1:0006:6bca:9502:eb69:11e7/64

 02BorderRouter        wpan0  thread   fd00:0000:fab1:0006:6a6a:f236:eb69:11e7/64
                                       2001:0db8:0001:0002:a651:3eff:fe92:6dbc/64

                       wlan0    wifi   fd00:0000:fab1:0001:6a6a:f236:eb69:11e7/64
                                                                      10.0.1.2/24
                                       2001:0db8:000a:000b:426c:38ff:fe90:01e6/64

     03WiFiNode        wlan0    wifi   2001:0db8:000a:000b:9aae:2bff:fe71:62fa/64
                                                                      10.0.1.3/24
                                       fd00:0000:fab1:0001:6b82:6e60:eb69:11e7/64
Each node has been added to the Weave fabric, and each interface has a new IPv6 address starting with fd00. To get more information on the Weave fabric, use the weave-state command:

$ weave-state

State Name: weave

NODES           Name       Weave Node Id    Pairing Code
        01ThreadNode    69ca9502eb6911e7          8ZJB5Q
      02BorderRouter    686af236eb6911e7          B5YV3P
          03WiFiNode    69826e60eb6911e7          L3VT3A

FABRIC     Fabric Id           Global Prefix
                fab1     fd00:0000:fab1::/48
Here is our updated topology, with Weave values in blue:



Weave fabric
There's a lot of new information in the Weave and Happy states. Let's start with the fabric from weave-state:

FABRIC     Fabric Id           Global Prefix
                fab1     fd00:0000:fab1::/48
Weave uses an IPv6 prefix of fd00::/48 for each node. Addresses in this block are called Unique Local Addresses and are designated for use within private networks such as a HAN. Combining that with the Fabric ID generates the Weave Global Prefix shown above.

Weave nodes
Each node in the Weave fabric is assigned a unique Node ID, along with a Pairing Code:

NODES           Name       Weave Node Id    Pairing Code
        01ThreadNode    69ca9502eb6911e7          8ZJB5Q
      02BorderRouter    686af236eb6911e7          B5YV3P
          03WiFiNode    69826e60eb6911e7          L3VT3A
The Node ID globally identifies a node in the Weave fabric. The Pairing Code is used as a "joiner credential" during the pairing process, and typically would be printed alongside a product's QR code.

For example, if you look at the QR code on a Nest Protect or a Nest Cam, you'll notice a 6-character string, often referred to as the Entry Key. This is the Weave Pairing Code.



Weave uses a combination of the Global Prefix, the Fabric ID, and the Node ID to create Weave-specific IPv6 addresses for each node and interface in the fabric.

Weave addresses
Note that there are four new IPv6 addresses in the Happy topology, all starting with our Weave Global Prefix of fd00:0000:fab1::/48.

NODES      Name    Interface    Type                                          IPs
   01ThreadNode        wpan0  thread   2001:0db8:0001:0002:3e36:13ff:fe33:732e/64
                                       fd00:0000:fab1:0006:6bca:9502:eb69:11e7/64

 02BorderRouter        wpan0  thread   fd00:0000:fab1:0006:6a6a:f236:eb69:11e7/64
                                       2001:0db8:0001:0002:a651:3eff:fe92:6dbc/64

                       wlan0    wifi   fd00:0000:fab1:0001:6a6a:f236:eb69:11e7/64
                                                                      10.0.1.2/24
                                       2001:0db8:000a:000b:426c:38ff:fe90:01e6/64

     03WiFiNode        wlan0    wifi   2001:0db8:000a:000b:9aae:2bff:fe71:62fa/64
                                                                      10.0.1.3/24
                                       fd00:0000:fab1:0001:6b82:6e60:eb69:11e7/64
Weave protocols use these addresses to communicate across the Weave fabric, rather than the standard IPv6 addresses assigned to each node.

Note: Nodes designated as access points and services in Happy are not part of the Weave fabric configuration—meaning that they do not receive Weave IPv6 addresses. See happy-node-add -h for more information on node types.

Weave network gateway
Weave nodes on a Thread network need to know where to exit that network. A Weave network gateway—typically on a Thread Border Router—provides this functionality.

In our sample topology, let's designate the BorderRouter node as the Weave network gateway:

$ weave-network-gateway ThreadNetwork 02BorderRouter
This command adds a route from all Thread nodes to the Weave fabric subnet (fd:0:fab1::/48) via the BorderRouter node's Thread interface (wpan0), which enables each Thread node to reach any Weave node beyond the Thread network. This is analogous to the happy-network-route command we used earlier, but specific to Weave fabric routes.

## Topology maintenance

What makes Happy so powerful is how it easily manages all setup and teardown of a simulated topology.

Save your Happy topology for later use:

$ happy-state -s codelab.json
This places a JSON file with the complete topology in your root ~/ folder. The JSON file is a copy of your current Happy state, which is found at ~/.happy_state.json.

Once saved, delete the current topology:

$ happy-state-delete
This deletes all network namespaces and related configurations found in the ~/.happy-state.json file. Check happy-state and weave-state to confirm an empty configuration:

$ happy-state

State Name:  happy

NETWORKS   Name         Type   State                                     Prefixes

NODES      Name    Interface    Type                                          IPs


$ weave-state

State Name: weave

NODES           Name       Weave Node Id    Pairing Code

FABRIC     Fabric Id           Global Prefix
To reload a saved configuration, use one of two commands:

happy-state-load — does not support Weave plugin
weave-state-load — supports Weave plugin
So if your topology includes Weave, always use the weave-state-load command so that the Weave fabric and associated configuration is applied.

Reload the saved Happy topology:

$ weave-state-load codelab.json
Note: During weave-state-load, you may get messages like this:

[HappyStateLoad] [01ThreadNode] HappyStateLoad: Route record to default_v4 via 02BorderRouter already exists.

[HappyStateLoad] [03WiFiNode] HappyStateLoad: Route record to default_v6 via 02BorderRouter already exists.

When Happy loads a saved topology, it applies all saved routes when bringing up each node, then tries to run the route-specific commands we did, like happy-network-route and weave-network-gateway. Since those routes are already set up, you get the messages above. They are nothing to worry about, all is well.

Check all states to confirm a successful loading:

$ happy-state

State Name:  happy

NETWORKS   Name         Type   State                                     Prefixes
  ThreadNetwork       thread      UP                       2001:0db8:0001:0002/64

    WiFiNetwork         wifi      UP                       2001:0db8:000a:000b/64
                                                                        10.0.1/24


NODES      Name    Interface    Type                                          IPs
   01ThreadNode        wpan0  thread   2001:0db8:0001:0002:eef6:a0ff:feca:6697/64
                                       fd00:0000:fab1:0006:6bca:9502:eb69:11e7/64

 02BorderRouter        wpan0  thread   fd00:0000:fab1:0006:6a6a:f236:eb69:11e7/64
                                       2001:0db8:0001:0002:5e53:bbff:fe05:484b/64

                       wlan0    wifi   2001:0db8:000a:000b:2e61:fdff:fed9:4fbc/64
                                       fd00:0000:fab1:0001:6a6a:f236:eb69:11e7/64
                                                                      10.0.1.2/24

     03WiFiNode        wlan0    wifi   fd00:0000:fab1:0001:6b82:6e60:eb69:11e7/64
                                                                      10.0.1.3/24
                                       2001:0db8:000a:000b:5e8e:c9ff:fed2:bdd1/64


$ weave-state

State Name: weave

NODES           Name       Weave Node Id    Pairing Code
        01ThreadNode    69ca9502eb6911e7          8ZJB5Q
      02BorderRouter    686af236eb6911e7          B5YV3P
          03WiFiNode    69826e60eb6911e7          L3VT3A

FABRIC     Fabric Id           Global Prefix
                fab1     fd00:0000:fab1::/48
A number of pre-defined topologies have been provided in the Happy repository, in both shell-script and JSON format. Find them at ~/happy/topologies.

OpenWeave also comes with select pre-defined Happy topologies for testing purposes. Find them at ~/openweave-core/src/test-apps/happy/topologies/standalone.

## How it works
Happy uses Linux network namespaces to simulate complex topologies. Typically, a network configuration applies across the entire Linux OS. Network namespaces allow you to partition network configurations so that each namespace has its own set of interfaces and routing tables.

Each node and network in Happy is a network namespace, while links between them are network interfaces.

For example, using our topology:



Let's see what namespaces Happy created for this:

$ ip netns list
happy004
happy003
happy002
happy001
happy000
Always use Happy commands to configure the topology via namespaces. They are the only way to safely modify your Linux machine's networking configuration. Do not modify it manually via ip commands unless you know what you're doing.

If you check the netns section of the Happy state JSON file, you can see what nodes and networks each namespace corresponds to:

$ happy-state -j | grep "netns" -A 5
"netns": {
    "01ThreadNode": "000",
    "02BorderRouter": "001",
    "03WiFiNode": "002",
    "ThreadNetwork": "003",
    "WiFiNetwork": "004",


Run-time logs
Commands issued to nodes are basic terminal commands executed from within each node's namespace. An easy way to see this is to enable Happy run-time logs.

Open a second terminal window and turn on logs, they will run continuously on this window:

$ happy-state -l
Go back to the first window and run a Happy ping:

$ happy-ping 01ThreadNode 02BorderRouter
Check the most recent log entries in the second terminal window. You should see a line like this in the logs:

DEBUG [Driver:CallCmd():416] Happy [happy]: > sudo ip netns exec happy000 ping6 -c 1 2001:0db8:0001:0002:5e53:bbff:fe05:484b
The happy-ping command is nothing more than Happy running the ping6 command in the happy000 namespace (01ThreadNode).

Enter a node
Use happy-shell to run non-Happy commands as if logged into one of the nodes (network namespaces):

$ happy-shell 01ThreadNode
root@01ThreadNode:#
Simulated devices are run within each namespace, and they only have access to the network configuration specified through Happy.

Check the interface configuration for the node. This will be different than your OS-wide configuration and should reflect what's listed in the Happy state:

root@01ThreadNode:# ifconfig
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:1 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:152 (152.0 B)  TX bytes:152 (152.0 B)

wpan0     Link encap:Ethernet  HWaddr ec:f6:a0:ca:66:97
          inet6 addr: fd00:0:fab1:6:6bca:9502:eb69:11e7/64 Scope:Global
          inet6 addr: 2001:db8:1:2:eef6:a0ff:feca:6697/64 Scope:Global
          inet6 addr: fe80::eef6:a0ff:feca:6697/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:32 errors:0 dropped:0 overruns:0 frame:0
          TX packets:26 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:2832 (2.8 KB)  TX bytes:2348 (2.3 KB)
Use exit to leave the node's namespace:

root@01ThreadNode:# exit

## Connect to a service
With an understanding of how Happy uses Linux network namespaces, you might now realize that it is possible to connect a simulated Happy network to the internet and access public addresses from within simulated nodes. This is useful for connecting your simulated devices to a real service (like the Nest Service over Weave).

The service in Weave is a cloud-based infrastructure that connects HAN nodes into a data model, provides remote access, and implements intelligent controllers to create a comprehensive ecosystem.

The service can be represented two primary ways with Happy:

As a simulated service in its own network namespace (Happy node)
As a real cloud service on the internet
Pre-defined topologies have been provided in ~/happy/topologies as an example of each service scenario.

Simulated service on a Happy node
Remove any existing Happy topologies:

$ happy-state-delete
Confirm an empty state with the happy-state and weave-state commands.

Load a pre-defined topology with access point (AP) and service nodes:

$ weave-state-load ~/happy/topologies/thread_wifi_ap_service.json


Check the Happy and Weave states to confirm the topology. In this topology, onhub is the AP, while cloud is the simulated service. Note both are connected to an Internet network of type wan:

$ happy-state

State Name:  happy

NETWORKS   Name         Type   State                                     Prefixes
     HomeThread       thread      UP                       2001:0db8:0111:0001/64

       HomeWiFi         wifi      UP                       2001:0db8:0222:0002/64
                                                                        10.0.1/24

       Internet          wan      UP                               192.168.100/24


NODES      Name    Interface    Type                                          IPs
   BorderRouter        wpan0  thread   2001:0db8:0111:0001:f624:13ff:fe4a:6def/64
                                       fd00:0000:fab1:0006:1ab4:3000:0000:0005/64

                       wlan0    wifi                                  10.0.1.2/24
                                       fd00:0000:fab1:0001:1ab4:3000:0000:0005/64
                                       2001:0db8:0222:0002:9e31:97ff:fe73:29f0/64

     ThreadNode        wpan0  thread   2001:0db8:0111:0001:c237:fbff:fecc:b082/64
                                       fd00:0000:fab1:0006:1ab4:3000:0000:0009/64

          cloud         eth0     wan                             192.168.100.3/24

          onhub        wlan0    wifi                                  10.0.1.3/24
                                       2001:0db8:0222:0002:3266:20ff:fe98:6ee2/64

                        eth0     wan                             192.168.100.2/24


$ weave-state

State Name: weave

NODES           Name       Weave Node Id    Pairing Code
        BorderRouter    18B4300000000005          AAA123
          ThreadNode    18B4300000000009          AAA123

FABRIC     Fabric Id           Global Prefix
                fab1     fd00:0000:fab1::/48
Weave tunnel
A Weave tunnel connects the Weave fabric to a service. This is a secure route that transfers IPv6 UDP messages between the HAN and the service. In this topology, the BorderRouter node is the Weave network gateway, which functions as the tunnel endpoint on the HAN.

Create the Weave tunnel:

$ weave-tunnel-start BorderRouter cloud
Recheck the Happy state. You should see a new tunnel interface with a Weave IPv6 address on the cloud node:

NODES      Name    Interface    Type                                          IPs

          cloud service-tun0     tun   fd00:0000:fab1:0005:1ab4:3002:0000:0011/64

                        eth0     wan                             192.168.100.3/24


You can now successfully ping between nodes on the Weave fabric and the service's Weave global prefix:

$ happy-ping ThreadNode cloud
[Ping] ping from ThreadNode to cloud on address
    fd00:0000:fab1:0005:1ab4:3002:0000:0011 -> 0% packet loss
Real cloud service on the internet
Remove any existing Happy topologies:

$ happy-state-delete
Confirm an empty state with the happy-state and weave-state commands.

Load a pre-defined topology with an access point (AP) node:

$ weave-state-load ~/happy/topologies/thread_wifi_ap_internet.json


In this topology, onhub is the AP. Check the Happy state. It's similar to the previous topology, without the Internet network and cloud node:

$ happy-state

State Name:  happy

NETWORKS   Name         Type   State                                     Prefixes
     HomeThread       thread      UP                       2001:0db8:0111:0001/64

       HomeWiFi         wifi      UP                       2001:0db8:0222:0002/64
                                                                        10.0.1/24


NODES      Name    Interface    Type                                          IPs
   BorderRouter        wpan0  thread   2001:0db8:0111:0001:ca3f:71ff:fe53:1559/64
                                       fd00:0000:fab1:0006:1ab4:3000:0000:0006/64

                       wlan0    wifi   2001:0db8:0222:0002:32e7:dfff:fee2:107a/64
                                       fd00:0000:fab1:0001:1ab4:3000:0000:0006/64
                                                                      10.0.1.2/24

     ThreadNode        wpan0  thread   2001:0db8:0111:0001:c2fb:97ff:fe04:64bd/64
                                       fd00:0000:fab1:0006:1ab4:3000:0000:000a/64

          onhub        wlan0    wifi                                  10.0.1.3/24
                                       2001:0db8:0222:0002:3a3c:8dff:fe38:999b/64


$ weave-state

State Name: weave

NODES           Name       Weave Node Id    Pairing Code
        BorderRouter    18B4300000000006          AAA123
          ThreadNode    18B430000000000A          AAA123

FABRIC     Fabric Id           Global Prefix
                fab1     fd00:0000:fab1::/48
Since each Happy node is a network namespace, they are partitioned from the public internet by default. Test this by entering a Happy node and pinging a public internet address. We'll use 8.8.8.8, one of google.com's IPv4 addresses.

$ happy-shell onhub
root@onhub:# ping -c2 8.8.8.8
connect: Network is unreachable
To connect the onhub node to the internet, it must be bridged to that interface on the Linux OS-level configuration.

Exit the node:

root@onhub:# exit
Determine the interface for your internet connection with the route command:

$ route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         192.168.1.0     0.0.0.0         UG    0      0        0 em1
Find the default route. This is the internet connection for your Linux machine. The Iface column indicates which interface is being used for that connectivity. In the example above, it's em1.

Use happy-internet to set up the bridge, using the interface for your default route. For the --isp flag, use the interface name without trailing numbers. In this example, it's em. If your default interface is eth1, the --isp flag would be eth.

$ happy-internet --node onhub --interface em1 --isp em --seed 249
Always use Happy commands to configure the topology via namespaces. They are the only way to safely modify your Linux machine's networking configuration. Do not modify it manually via ip commands unless you know what you're doing.

There will be no visible change in the happy-state output, but the onhub node should have internet connectivity. Let's go back into the node and check:

$ happy-shell onhub
root@onhub:# ping -c2 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=56 time=1.81 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=56 time=1.81 ms

--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 1.814/1.816/1.819/0.042 ms
Success!



DNS
Happy nodes do not have built-in DNS capabilities. If you try to ping google.com, it fails:

root@onhub:# ping -c2 google.com
ping: unknown host google.com
Fortunately, Happy provides support for DNS. Exit the node and find the DNS servers for your Linux machine. Make sure to use the appropriate default interface:

root@onhub:# exit
$ nmcli dev list iface em1 | grep domain_name_servers
DHCP4.OPTION[13]:                       domain_name_servers = 172.16.255.1 172.16.255.153 172.16.255.53
Use these DNS addresses with happy-dns:

$ happy-dns 172.16.255.1 172.16.255.153 172.16.255.53
Now try pinging google.com from within the onhub node:

$ happy-shell onhub
root@onhub:# ping -c2 google.com
PING google.com (64.233.191.113) 56(84) bytes of data.
64 bytes from ja-in-f113.1e100.net (64.233.191.113): icmp_seq=1 ttl=46 time=36.9 ms
64 bytes from ja-in-f113.1e100.net (64.233.191.113): icmp_seq=2 ttl=46 time=37.0 ms

--- google.com ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 36.978/36.995/37.013/0.193 ms
Exit the onhub node when you're done:

root@onhub:# exit
Weave tunnel
Like the simulated service, a Weave tunnel must be set up between the simulated HAN in Happy and the service. With a real cloud service, use the IP address or URL of the service in the tunnel setup. For example:

$ weave-tunnel-start BorderRouter mycloud.service.com

## Cleaning up
It's important to always clean up Happy topologies when you're done with them, to avoid issues with your Linux network configuration.

Always use Happy commands to clean up your topologies. They are the only way to safely modify your Linux machine's networking configuration. Do not modify it manually via ip commands unless you know what you're doing.

If you enabled DNS support in your topology, remove it by rerunning that command with the -d (delete) flag first. This must be run before any Happy nodes are removed, to ensure the network configuration is properly updated.

$ happy-dns -d 172.16.255.1 172.16.255.153 172.16.255.53
Next, delete the Happy state:

$ happy-state-delete
Occasionally, some state files may remain after a state deletion. If you run into problems and Happy is not working as expected, delete the state with happy-state-delete and then use the following commands to force any remaining clean up:

$ ip netns | xargs -I {} sudo ip netns delete {}
$ rm -rf ~/.*state.json
$ rm -rf ~/.*state.json.lock
Your machine should be back to its normal network configuration.

## Official

https://codelabs.developers.google.com/codelabs/happy-weave-getting-started/#1