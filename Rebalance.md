[Back To Maintenance Page](Maintenance.md)

# Rebalancing Files #

**note** This is a new feature (as of 2.40). It may change or contain bugs. We always do the best we can to ensure MogileFS is stable, but be careful.

In 2.40+ MogileFS has a rewritten drain/rebalance system. You define a set of
rules and it will select devices to pull files from, and devices to put files
onto.

So if you're looking to retire some devices, or have added new hosts and wish
to move some files onto it, this is how you do that.

## When to Run Rebalance ##

Rebalancing isn't a required operation of mogile. However it is good practice
to pay attention to your device storage and make decisions on where your files
should be. If you add three new empty hosts, it's often better to shuffle
existing files onto them, freeing up space across all of your hosts to
distribute new files (which may be accessed more frequently than older ones).

MogileFS replication also enjoys having lots of options to replicate files
toward. So keep it happy :)

# Rebalance Policies #

The new rebalance/drain system works via policies. You define a string of
options and the system evaluates the options and makes a decision. As of this
writing rebalance is under construction, so the options here may not be all of
the options available.

To show the rebalance settings:
```
$ mogadm rebalance settings
             rebal_policy = from_percent_used=95 to_percent_free=50 limit_type=device limit_by=size limit=5g fid_age=old
```

And to set them:
```
$ mogadm rebalance policy --options="from_hosts=3 to_percent_free=50"
```

When you first start a rebalance, mogile will discover and save the list of
source devices. However, every few seconds it will re-evaluate the list of
destination devices. This is both for avoiding a ping-pong state and tpo
always find the best possible candidates for a destination.

## Policy Options ##

(and their defaults)

```
    # source
    from_hosts => [],           # host ids (not names).
    from_devices => [],         # device ids.
    from_percent_used => undef, # 0.nn * 100
    from_percent_free => undef,
    from_space_used => undef,
    from_space_free => undef,
    fid_age => 'old',           # old|new
    limit_type => 'device',     # global|device
    limit_by => 'none',         # size|count|percent|none
    limit => undef,             # 100g|10%|5000

    # target
    to_hosts => [],
    to_devices => [],
    to_percent_used => undef,
    to_percent_free => undef,
    to_space_used => undef,
    to_space_free => undef,
    not_to_hosts => [],
    not_to_devices => [],
    use_dest_devs => 'all',     # all|N (list up to N devices to rep pol)
    leave_in_drain_mode => 0,
```

`from_(percent|space)_*`: Pull fids from devices "at least this x". At least
this much space used, this much percent free. The space options are expressed
in megabytes.

`to_(percent|space)_*`: Same as above, except used in limiting possible
destination devices.

`(from|to)_hosts`: Given a comma separated list of host ids, choose all devices

`(from|to)_devices`: Directly specify which devices you want to pull from or
drop files onto. Other options may further reduce this list (_**percent\_used,
etc). Note: This is a comma seperated list of the device ids, e.g. from\_devices=199,201,233**

`not_to_*`: filter out specific devices or hosts from the destination._

`fid_age`: Defines whether rebalance will choose "old" (ascending numbered)
fids or "new" (descending numbered) fids from the device first. Since MogileFS
uses an incrementor for the fid, files are naturally ordered by age. In some
setups "old" files may be accessed less often than new ones, which can
influence your rebalancing decision.

`limit_type`: (global|device) whether or not the specified limit is applied
globally (drain 5000g in total from any of these devices), or per device (pick
12 devices and drain 10g from each)

`limit_by`: (size|count|percent|none) defines what the limit is. **note** as of
writing "percent" is unimplemented. You can limit by 'count', which would be a
number of files to move, by 'size', a specific number of bytes to copy, or
'none', which means to kill all files from the devices.

`limit`: The limit as defined by you above. The 'size' limit is expressed in bytes by default, but takes a human modifier (500m, 10g, 13t, etc)

`use_dest_devs`: (all|N) after applying all filters, you may have any number
of destination devices. A handful, dozens, hundreds. This limits the amount of
devices that replication will later consider. It's mostly an optimization, if
you have any devices you might want to set this to some reasonable number.

`leave_in_drain_mode`: (0|1) In previous versions setting a device to 'drain' meant
that mogile would harass the device automatically and remove files. Now it
simply means "don't put new files here". While rebalance is working on a
device, it is set from alive and into drain mode. If you wish to emulate the
old drain behavior, you set this value to 1. Setting no limit and enabling
this will remove all files from a device and not allow new ones to be added
again.

# Running a Rebalance #

As noted above, create your policy. You can review it via `mogadm rebalance settings`

## Testing ##

```
$ mogadm rebalance test
Tested rebalance policy...
Policy: etc

Source devices:
 - 100
 - 102
 - 103
 - 104
Destination devices:
 - 156
 - 157
 - 158
 - 159
```

Before starting a rebalance, you should review what devices the policy would
match.

Hopefully future versions will display more information about the devices, but
for now you may match the lists up against the output of `mogadm check`
mentally :)

## Starting ##

```
$ mogadm rebalance start
$ mogadm rebalance stop
$ mogadm rebalance reset
```

Rebalance will pause when stopped, but existing entries in REBAL\_QUEUE will continue to run. You can view the queue with `mogstats --stats="general-queues"`. It may be advisable to wait for this to finish before starting again.

To restart your rebalance from scratch, either change the policy or run a `mogadm rebalance reset` while stopped.

## Watching ##

```
$ mogadm rebalance status
Rebalance is running
Rebalance status:
             bytes_queued = 126008251219
           completed_devs = ,102,125,151,148
              fids_queued = 519021
             sdev_current = 119
             sdev_lastfid = 54646960511250969
               sdev_limit = 2840763873
              source_devs = 108,115,103,113,152,142,107,141,100
```

The status output for rebalance is simply a dump of its internal state, along
with some counters it keeps. The status is updated every few seconds after the
job master runs.

`sdev_current` is the device it's working on.

`sdev_limit` are the remaining bytes (or number of files) rebalance is
attempting to move.

`fids_queued` is the global count of how many fids have been queued since
starting.

`bytes_queued` are the number of bytes moved since starting.

The state will stick around after rebalance has finished running, so you may
look at the final state.

It's generally a good idea to watch mogile's syslog output during a rebalance.
If it runs into fids it cannot rebalance for some reason, the information is
sent to syslog (or !watch if you telnet to a tracker).

## Optimizing ##
Most of the optimizations for an FSCK apply to a rebalance so see: http://code.google.com/p/mogilefs/wiki/FSCK#Tuning_FSCK for tips on speeding up the process.

Basically increase the number of replicate jobs across your trackers.
```
telnet trackername trackerport
!want 5 replicate
```

There is a configurable setting: queue\_rate\_for\_rebal which defaults to 60 (in Mogilefs 2.45). Most people can just leave it as is. If adding mor e replicate processes stops helping and load across trackers/database is still low you may want to look into tweaking this.


---

# Rebalance Examples #

## Make everything roughly even ##

As of this writing there're some missing shortcuts for evening out your file
distribution. There're two approaches to doing this:

One is to calculate how much disk space to move from each device on the
"fuller" devices, and where to put them ie:

`from_percent_used=90 to_percent_used=10 limit_type=device limit_by=size limit=5g`

If you have two hosts with four full disks each, and two new hosts with empty
disks, the above would move 40g of data onto the new hosts.

The other approach is to run rebalance over and over until `mogadm rebalance test` doesn't pick any new devices to work on.

`from_percent_used=51 to_percent_free=51 limit_type=device limit_by=size limit=1g`

The above will take files off anything > 51% full, and move 1g from each
device toward anything at least 51% empty. The theory is if you run this
enough times it should trend towards even. Be mindful; if your entire
cluster is more than 50% full, you could end up re-running rebalance forever.


---

# How Rebalance Works #

## Filter Devices ##

Simply put, the above document tells you how to build a string which filters
the list of devices and selects which ones should or shouldn't be drained
from, or should or shouldn't be replicated to.

Once the devices are selected, rebalance will run them through one at a time
and queue up fids for the replication workers to actually rebalance.

## Internally Rebalance ##

Below is a high level example of the flow a replicate worker has for rebalance. This is
useful to know in case of seeing "would\_worsen" errors on small clusters.

  * fid 5 has 3 copies. one on each of dev 31, 32, 33.
  * rebalance decides to pull the fid off of dev33.

Rebalance tells the replication code "replicate FID 5, and pretend you don't
have this copy on dev33, and you can't put the FID on dev33"

This is a constraint that rebalance can never accidentaly leave a fid worse off than it was before, if anything goes wrong it should be "too\_happy" and not sad. This also deals with being able to move fids which have only one valid copy. Otherwise the drain code would nuke the fid.

  * Now that it has 4 copies, the rebalance code deletes the one from dev33

  * It should then have 3 copies, and is now balanced away from dev33.