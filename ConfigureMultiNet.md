# Configuring MogileFS for Multiple Networks


**Note:** this guide refers to version 2.35 and higher, and also requires modules which may not be on CPAN as of this writing.

# MogileFS and Multiple Networks/Datacenters #

MogileFS understands multiple datacenters in terms of multiple available networks. Which means:

  * Given two storage nodes with IPs 10.5.0.1 and 10.6.0.1.
  * You may configure mogile to understand 10.5/16 and 10.6/16 are different network "zones"
  * Which then may be located in distinct datacenters, cabinets, floors, racks, etc.
  * With proper configuration, you may state how many copies of a file you want to be in each zone.

It does **not** presently have any native ability to run its database or trackers in multiple locations. Trackers may be run on either side of a network, and with the `ZoneLocal` plugin they will attempt to serve clients with local copies, and replicate using local copies. However, all trackers must still point to the same database to retrieve their data.

# Installing Prerequisites #

Install 'MogileFS::Network' from CPAN, onto your tracker servers. This provides replication policies for `MultipleNetworks` and `HostsPerNetwork`, as well as providing a `ZoneLocal` plugin for trackers.

# Configuring #

## Decide on a Replication Policy ##

### `MultipleNetworks` ###

This replication policy does not require explicit configuration. Define your network zones, and simply set the class policy to "`MultipleNetworks()`". Replication will then attempt to ensure copies exist on as many networks as possible. Ideally you use this with a mindevcount of 3 or higher.

### `HostsPerNetwork` ###

A more powerful version of `MultipleNetworks`, this allows you to specify exactly how many copies should be in each zone. For example. if you have zones "near", and "far". You want two copies in near, and one copy in far, you should configure this as "`HostsPerNetwork(near=2,far=1)`".

## Configuring Zones ##

  * `mogadm settings set network_zones near,far`
  * `mogadm settings set zone_near 10.5.0.0/16`
  * `mogadm settings set zone_far 10.6.0.0/16`

... and that's it.

## Configuring `ZoneLocal` Plugin ##

Once you have zones configured, and MogileFS::Network is installed on all trackers, you have the option of enabling a tracker plugin called `ZoneLocal`. This will tell the tracker to figure out which zone it exists in, as well as attempt to discover which zone remote clients exist in. It will then attempt to use paths that are available on its "local" network when returning path data or replicating new copies of a file.

To enable this, simply add two new lines to your mogilefsd.conf
```
local_network = 10.5.0.0/16
plugins = ZoneLocal
```

local\_network should be the network which clients and nodes will think this tracker is coming from. More simply said, it's just the network that the tracker is on. If its IP address is 10.5.0.50, the local\_network would be 10.5.0.0/16.

For your first time setting this up, you should consider running one tracker in the foreground (not daemonized), or carefully monitoring via !watch to ensure it's happy.

## Configuring Class Replication Policies ##

The last part is to configure class policies. MogileFS allows you to decide on multiple network policies on a per-file basis via classes.

```
mogadm class list
 domain               class                mindevcount   replpolicy  
-------------------- -------------------- ------------- ------------
 toast                byhost                    2        MultipleHosts()
 toast                default                   2        MultipleHosts()
 toast                four                      4        MultipleHosts()
 toast                fourbynamenet             1        HostsPerNetwork(near=2,far=1)
mogadm class add toast twoontwonets --replpolicy "HostsPerNetwork(near=2,far=2)"
mogadm class modify toast twoontwonets --replpolicy "HostsPerNetwork(near=3,far=3)"
```

# Example Setup #

Given a network of: 10.10.0.0/16 (`10.10.*.*` addresses)

All of your machines are configured to have a netmask of 10.10.0.0/16

When assigning IP addresses to machines, pick them from 10.10.5.0/24, ie:

web1: 10.10.5.1 (netmask 255.255.0.0 or /16)
web2: 10.10.5.2
tracker1: 10.10.5.3
tracker2: 10.10.5.4
storage node 1: 10.10.5.5
storage node 2: 10.10.5.6

Then, your backup node:

storage node 3: 10.10.8.1 (netmask 255.255.0.0 or /16)

So all of your main servers fit in 10.10.5.0/24
Your backup servers are in 10.10.8.0/24

Then in MogileFS zones, you configure:

`near=10.10.5.0/24 far=10.10.8.0/24`

Now, when "web1" makes an API request to "tracker1", tracker1 sees that web1
is in the "near" zone of 10.10.5.0/24, because "web1" has an IP address of
10.10.5.1

"tracker1" now looks at what storage nodes are also in "near", which matches
"node1" and "node2" as they have IP addresses of 10.10.5.5, and 10.10.5.6,
which are also within 10.10.5.0/24

Since "node3" is 10.10.8.1, it matches zone "far" of 10.10.8.0/24 and is not
considered for reads.

If you set a `replpolicy` of:

class toast: `HostsPerNetwork(near=2,far=1)`

"tracker1" and "tracker2" will always put two copies of a file with class
"toast" into the "near" zone, and one in the "far" zone. In this example, that
translates to one copy on "node1", one copy on "node2", and one copy on "node3"

# Wrapping Up #

When you're done setting it up, configuring it, and modifying some classes, keep in mind that it does not automatically re-replicate any files. If you're modifying a replication policy and have existing files, you'll want to run a FSCK for mogilefs to figure out what it needs to do to make these new policies happy. See the [FSCK](FSCK.md) guide for more details.