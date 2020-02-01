# Upgrading MogileFS #

(note: this only covers upgrades within the 2.x+ series. If you have a 1.x install, consult the mailing list).

Upgrading mogilefs requires a little thought but is mostly seamless. Most releases are very backwards compatible with others. However it is not recommended to run clusters of mogilefs trackers on different versions for an extended period of time. If you upgrade some but not others, plan on catching up the others within a few hours or days, and don't leave a gap of many releases where possible.

## mogdbsetup Notes ##

mogdbsetup is a utility that will either install or upgrade a mogilefs schema. It does not remove existing data, but regardless be sure you have good backups before running the tool. You should also be familiar with the CHANGES list for details on what any schema changes are and how long they might take to run.

Schema bumps are rare, and usually affect tables that have less than a few dozen rows in them. In the case of a large table needing modification, mogilefs tries to be more accommodating.

A tracker will only start on a schema version it is familiar with. If you run mogdbsetup and bump a schema revision, trackers running code prior to this release will refuse to restart, unless you provide '--no\_schema\_check'

You may also run mogdbsetup with '--no\_schema\_bump' to have it run the upgrade but not bump the schema yet (you should **never** leave your cluster like this for long).

## Upgrading a Tracker ##

Most upgrades are easy:

  * Install new version of mogilefs-server (and any dependencies that might be bumped)
  * Run mogdbsetup. This will upgrade your schema if a new one is required. (NOTE: If there is a schema bump, your old tracker software will refuse to start again, once mogdbsetup is finished.)
  * Stop the tracker
  * Start the tracker
  * telnet to the tracker and run '!watch', test your application(s) and look for any bad errors.

## Upgrading Multiple Trackers ##

Ideally you have more than one tracker. Even more ideal is if you have a "staging" tracker, or can remove one from production rotation during an upgrade test.

  * Read the 'mogdbsetup' section below to be familiar with options if a schema upgrade is needed.
  * Run mogdbsetup
  * Stop the tracker
  * Start the tracker
  * telnet to the tracker, run '!watch', and test your application(s)
  * If you can't easily directly test your application(s) against the tracker, try uploading some files via mogtool.
  * Also, on a another tracker, run '!recent' to examine a few recent commands, and copy/paste some get\_paths requests to the new tracker. You should get working results from either.
  * Let the new version "burn in" for a few hours or a day or two without problems, then upgrade the others.

### For the Paranoid ###

  * Plan for a downtime.
  * Stop all your trackers.
  * Run mogdbsetup.
  * Upgrade, restart, and test all your trackers.
  * Resume running your application(s)

### For The Confident ###

If you've reviewed the requested schema change and believe it to be safe (or the release notes say it's backwards compatible), follow the "Upgrading (A/Multiple) Tracker(s)" instructions above. Upgrade one tracker first, test, then upgrade the rest.

## Tracker Upgrade Notes ##

As of 2.33, mogilefs doesn't have a "graceful stop" feature. I really wish I'd remembered this before running `shipit`. Ugh.

This means that user uploads can fail if you stop the tracker. Since a user upload is:
```
client -> tracker (create_open)
client -> storage node (PUT's file)
client -> tracker (create_close)
```
... which can happen on separate trackers, in separate processes, at different times. The safest method of stopping mogilefs is to use a firewall like iptables to block new connections from reaching the tracker, waiting a few minutes, then stopping the tracker. Like so:
```
/sbin/iptables -I INPUT -i eth0 -p tcp --dport 7001 -m state --state NEW -j REJECT
```
Once you start your tracker you can resume traffic by repeating the same command with a DELETE request to iptables:
```
/sbin/iptables -D INPUT -i eth0 -p tcp --dport 7001 -m state --state NEW -j REJECT
```
... you can incorporate this into your init scripts if you'd like. Newer releases will hopefully have a !gracefulshutdown command, which will stop accepting new connections and wait a while for existing ones to die off.


---


# Upgrading Mogstored #

Mogstored is the storage component of mogilefs. You use it either for monitoring, for uploads, for reads, depending on how you wish to wire it up. You typically use it in some form. Mogstored isn't edited as often as the tracker, but look for notes in the CHANGES file to see if it's worth upgrading your mogstored nodes.

Upgrading is simple :

  * Install the new mogilefs-server
  * Restart mogstored
  * Watch mogilefs (mogadm check, !watch) to see if it still works okay.


---


# Notes for Upgrades Between Specific Revisions #

## 2.70 ##

Major rewrite of the monitor worker to run device checks in parallel. Could
make a huge difference for people with many devices, or a mix of slow and fast
devices. Other workers are updated to use HTTP connections more efficiently, or run
some parts in parallel (fsck size checks, etc). See CHANGES for full details.

There are no DB schema changes in this release. It should be possible to test
the new release on part of your cluster. For example, if you have 3 trackers
that are on 2.68, you can upgrade one to 2.70 and run it for a few days to
compare performance.

Do not mix 2.70 with versions older than 2.68, however.

Note: Upgrading may increase concurrent file/socket usage. You may need to adjust the number of available open files/sockets to the mogilefs process if your prior setup did not already do so.

You can increase file/socket limits by placing a file in /etc/security/limits.d/ with the following contents. The below is just an example and may or may not be appropriate for your environment/setup.

```
*	soft	nofile	99999
*	hard	nofile	99999
```

Then be sure to restart the tracker process after the change is in place.

Note 2: You might need fewer workers in the newer release (especially delete workers)


## 2.68 ##

Many optimizations. Fix bug for device drain which could cause
over-replication in some circumstances. Bugfixes. See CHANGES for full
details.

## 2.67 ##

nginx is now an available backend for mogstored.  Several bugfixes, see
CHANGES for details.

## 2.66 ##

Mostly bugfixes. See CHANGES for full details.

## 2.65 ##

Bugfixes and rewrite of Reaper worker to improve DB efficiency when marking a
drive 'dead'. Postgres uses faster locking as well. See full CHANGES file for details.

## 2.64 ##

Small bugfixes and improvements to long standing issues (marking hosts as
'down' now counts its devices as 'down')

## 2.62 -- 2.63 ##

Small bugfix releases to correct Postgres issues with 2.61

## 2.61 ##

Many small bugfixes and many new automated tests. See CHANGES file for full
detail.

## 2.60 ##

Upgrades to schema version 15. Checksum release adds a new table. Checksums are off by default. Enabling checksum will store one extra row of information per fid stored in the database. See [Checksums](Checksums.md) for more information about this feature.

## 2.59 ##

Fix for postgres databases introduced with 2.58

## 2.58 ##

Fixes major issue with master-less read only mode not working in all
scenarios. Reduces database UPDATE's to the device table.

## 2.57 ##

Minor patch over 2.56 to work better with the new MySQL slave code.

## 2.56 ##

Many important fixes, removal of mogdeps, and MySQL slaving has been
refactored. If a master database goes down, MogileFS may still serve requests
from a slave. With multi-datacenter configurations, trackers will attemp to
use "local" slaves.

Also fixes a critical hanging bug from 2.55

**NOTE** If you use MySQL slaves, ZoneLocal requires slaves are configured as IP
addresses instead of with hostnames. This restriction may be lifted in the
future.

**NOTE** You must upgrade MogileFS::Network to the latest version if you want
local slave usage to work.

If you prefer to not sort slaves via ZoneLocal or cannot use IP addresses, set
the server setting "slave\_skip\_filtering" to "on" via mogadm

## 2.55 ##

Some important bugfixes see CHANGES for details. **WARNING** has a tracker bug where replciation/fsck
will hang until all trackers have ben restarted. Skip and use 2.56 or greater.

## 2.54 ##

Bunch of small fixes. See the CHANGES file for details.

## 2.53 ##

**important bugfix release**. If you're on 2.50+, you should upgrade. Otherwise
modifying mogile hosts will fail.

## 2.52 ##

Minor release. Upgrades to schema version 14, which is a trivial size change
on the device table, allowing devices larger than 16TB to be supported.

Also fixes some potential slowdowns with the way file sizes are checked. See
CHANGES file for full details.

## 2.51 ##

Minor release for 2.50. Recommended if you are running 2.50.

## 2.50 ##

No schema changes. Some postgres fixes. MogileFS::Utils 2.20 and higher
recommended.

Major rewrite of internal host/device metadata caching. Should reduce latency,
some DB load. Keep an eye out for bugs, but code has been heavily tested as
always.

## 2.46 ##

Some bugfixes. One fixing a critical bug only in 2.45. Another fixing more
postgres functionality.

## 2.45 ##

**Skip this release, due to a bug in an adjustment of how database handles were validated.**

Many bug-fixes, as well as changes to support the new command-line utilities.

If you were using the `list_fids` command to synchronize files outside of
mogilefs, the usage has changed slightly. It now takes a starting fid and a
count, instead of a from and to fid. The former would break as soon as a gap
in fids were found.

A bug was fixed where create\_open would allow you to upload a file to any
device, disregarding the list of devids create\_open actually returned. Now
it's strict about this, which may catch you if your client has bugs with
picking a device id to upload to.

No schema changes, should be an easy upgrade.

## 2.41 - 2.44 ##

All bugfix releases, no schema changes.

## 2.40 ##

Significant feature update and a schema bump. You will have to use mogdbsetup to bump your schema version. The schema is backward compatible, so it may be done to
an online system. Be wary that it ALTERs file\_to\_queue, which should normally
be very small. If you have a lot of queued work, however, you may want to wait until file\_to\_queue is empty.

Drain/Rebalance have been rewritten to be far more reliable and flexible. May
still be buggy in this release, so be careful.

See [Rebalance](Rebalance.md) for more information.

## 2.37 ##

Bugfix release. See main changelog, probably worth upgrading to as it fixes an
issue with HTTP DELETE (rebalance/drain/delete/etc).

## 2.36 ##

Bugfix release. Just fixes a critical bug in 2.35, and other minor bugs.

## 2.35 ##

**NOTE**: Relase has a critical bug with file creation. skip this release and use 2.36

Bugfix release. Old replication code has been completely removed. If you have old\_repl\_compat or no\_unreachable\_tracking options configured in your mogilefsd.conf, the tracker may complain on restart. These options haven't been necessary for over a year.

## 2.34 ##

Bugfix release. Trivial if coming from 2.33

## 2.33 ##

Easy upgrade. Don't run them mixed if you expect the fsck or replicate updates to work all the time (they won't fail, but sometimes you'll get old behavior). mogstored now no longer depends on Gearman.

## 2.32 ##

If your 'fid' columns (in tempfile, file, file\_on, fsck\_log, etc) are not BIGINT or better, you should consider ALTER'ing them. Installs will now default to the larger size, to avoid surprising people with upload failures after 4 billion uploads. It always happens sooner than you think.

## 2.30 ##

MogileFS 2.30 introduced new internal queue concepts, and adds new delete worker code. The schema bump is simple and can be done online. Running trackers < 2.30 with > 2.30 isn't recommended for any long period of time.