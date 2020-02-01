[Back To Maintenance Page](Maintenance.md)

# MogileFS Filesystem Check #

MogileFS, being a large distributed HTTP-backed data store, does not have a traditional 'fsck' component. It does have a mechanism for walking and verifying the contents of the entire filesystem in a parallel, online, asynchronous fashion.

The MogileFS fsck will, by default, walk across every FID stored and verify that:

  * The FID is happy (but not too happy) with its replication policy (ie: has 3 copies across multiple hosts).
  * That each copy of the FID is stored where it's supposed to be, exists, and is the correct length.
  * If a file has gone missing (no paths work), it will attempt to find the FID on any device.
  * Checks devcount cache, and some other misc minor notes.
  * Note that it doesn't yet confirm file checksums.

The FSCK is strictly brute force. You cannot focus it to a particular domain, class or prefix of files. However, each release of MogileFS is adding small improvements to the FSCK code, so check the latest code to see what's been added and updated.

## When to Run a FSCK ##

While you should not need to run a FSCK constantly, or normally, it's healthy to run one occasionally, or after major events. Major software errors, notable upgrades, power outages, etc. All good excuses for running a FSCK. Also, if you edit a class replication policy (add or remove replicas), the changes will not take effect until a FSCK has run.

FSCK can repair bugs in older versions. It can also help recover from situations where you don't have enough unique storage hosts to satisfy replication for a while. FIDs will end up on multiple devices, just not on enough hosts to be happy.

------

# Running a MogileFS FSCK #

## Kicking Off a FSCK ##

fsck is controlled via `mogadm`.

```
$ mogadm fsck 
Help for 'fsck' command:
 (enter any command prefix, leaving off options, for further help)

  mogadm fsck clearlog                               Clear the fsck log
  mogadm fsck printlog                               Display the fsck log
  mogadm fsck reset [opts]                           Reset fsck position back to the beginning
  mogadm fsck start                                  Start (or resume) background fsck
  mogadm fsck status                                 Show fsck status
  mogadm fsck stop                                   Stop (pause) background fsck
  mogadm fsck taillog                                Tail the fsck log
```

When starting a fsck for the first time, simply run 'mogadm fsck start'. After a few moments, it should start running. Run `mogadm fsck status` to watch its progress.

## FSCK options ##

You can start a fsck with a few options. If you want to run it against only newer files, you can tell it which FID number to start at, or tell it to only do a replication policy check without checking the state of files on disk.

```
mogadm fsck stop
mogadm fsck reset --startpos=5000 --policy-only
mogadm fsck start
```

## Monitoring a FSCK Run ##

If you like watching grass grow, fsck monitoring is perfect for you!
```
    Running: Yes
     Status: 55252778 / 75053798 (73.61%)
       Time: 791m (1164 fids/s; 19801020m remain)
 Check Type: Normal (check policy + files)

 [num_GONE]: 1
 [num_NOPA]: 1
 [num_POVI]: 365
 [num_REPL]: 365
 [num_SRCH]: 1
```
You may periodically run `mogadm fsck status` to monitor progress of a fsck. Note that the status information is slightly weird since version 2.30. 2.33 and higher have much improved status output, however keep in mind that once a fsck has switched to "Running: No", the fsck can still be filtering FIDs for a few minutes afterwards, while internal queues drain.

You can examine the results of the fsck with `mogadm fsck printlog`, or watch output while it's running with `mogadm fsck taillog`

## Tuning FSCK ##

In the 2.30+ versions, FSCK has gotten many tunable improvements. Previous fsck would run from exactly one worker process on a single tracker. If you had hundreds of millions of files, it could take over a month to run. Now that it's distributed, it can run as fast as you have resources available to run with.

It's a good idea to watch tracker logs (via syslog or running !watch while telnet'ed to a tracker), and look for timeout errors. If you start getting a lot of those, your cluster is running too hot.

### Number of FSCK Workers ###

The easiest tunable is simply the number of parallel fsck workers you have.
```
telnet trackerhost 7001
Trying 172.16.151.10...
Connected to 172.16.151.10
Escape character is '^]'.
!jobs
[...]
fsck count 1
fsck desired 1
fsck pids 32664
.
!want 5 fsck
!want 5 fsck
Now desiring 5 children doing 'fsck'.
.
```
Slowly increase the number of fsck workers while monitoring load across your trackers, database, and storage cluster. The trackers will slowly ramp up how much work it fetches from the fsck queue at once, so it could take a few minutes for them to ramp up. However, if you're adding more fsck workers and the fsck isn't running any faster, you probably need to tune a few other settings.

### FSCK Speed Settings ###

FSCK has a number of queue management options, which default relatively low to avoid adding too much load to small setups. If you have 25+ fsck workers (and your cluster isn't overloading!) you might need to tune these in order to get it to run faster.

```
$ mogadm settings list
     internal_queue_limit = 500
      queue_rate_for_fsck = 1000
      queue_size_for_fsck = 20000
[etc]
```
If you have not adjusted these values from the defaults, they will not display here. It's a little hard to discover what the defaults are until this is fixed (you can look in `JobMaster.pm`), but in the meantime the above are the defaults for 2.33 through 2.45

`internal_queue_limit` is the number of FIDs a tracker will fetch from the fsck database queue at once. Fsck workers will fetch chunks of FIDs internally from this queue. If you look at a tracker with the '!stats' command, you will see various variables like `work_queue_for_fsck 0`. If you are actively running a fsck with many workers, and this stat is often very low or 0, increasing the `internal_queue_limit` will allow it to go faster. Be wary of increasing it too high.. A few thousand is probably all you need. There is a slow internal ramp up, so be patient. You might also need to tune the values below to keep this fed.

`queue_size_for_fsck` is the number of FIDs that fsck will leave queued in the database for trackers to pick up and send to their workers. The higher this is the more disk space you waste, but if the queue is constantly zero, doubling or tripling the limit can help the queue stay ahead of demand.

`queue_rate_for_fsck` is the number of FIDs that fsck will inject into the queue per cycle (every other second-ish), if the queue is below the limit. Setting the variable too high can cause too much DB load, but too low and the trackers can get ahead of the queue. The total number of FIDs that can be queued at once could be up to `queue_size_for_fsck + queue_rate_for_fsck`.

## Interpreting Results ##

Fsck will log a bunch of obscure error codes as it finds and handles problems.

  * NOPA: FID has no usable paths, and is likely dead.
  * POVI: FID violates its class replication policy. This can mean too few, or too many copies.
  * MISS: FID is missing at least one copy that is supposed to exist.
  * BLEN: Copy of FID does not have the correct file length
  * GONE: Attempted to find any working, correct, copies of FID, but was not able to. File is dead. If you get these errors, you should take a deeper look at the FID and your app to find out why it might have gone missing.
  * SRCH: FSCK has started an exhaustive search for a missing file.
  * FOND: FSCK has recovered a FID that was thought to be lost.
  * REPL: FID has been scheduled for replication to fix a policy violation.
  * BCNT: FID's 'devcount' cache is incorrect.
  * BSUM: Database checksum didn't match on-disk contents
  * NSUM: Database checksum was missing from a class which requires checksums
  * MSUM: (Only for fsck\_checksum=$HASH users) multiple checksums were calculated and MogileFS did not know which is correct.

If you have a new enough version of MogileFS::Utils installed, you may take
fids noted in the FSCK logs and run them through `mogfiledebug`. This utility
will trawl everything it can find out a particular fid, which will help give
you ideas on what went wrong. If nothing else, it gives great output you can
use to get help on the mailing list or IRC.

## Cleaning up Between Runs ##

`fsck status` will count violation summaries from the log. Which means if you don't clear the log in between runs you will see more failures than there actually are. Also, the log file can get huge and take a lot of space on your database. So once you have looked through the log entries (or copied them into a file somewhere), it's a good idea to reset it.

```
mogadm fsck status
(confirm that it isn't running, and hasn't been for a few minutes at least)
mogadm fsck clearlog
mogadm fsck reset
```