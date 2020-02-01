[Back To Maintenance Page](Maintenance.md)

# Understanding the Job Workers #

MogileFS has a number of job workers per tracker, as noted in the [HighLevelOverview](HighLevelOverview.md). Most of these jobs can be scaled up or down at runtime to ramp up processing speed.

## Query Workers ##

Query workers are the only directly app/client interfacing workers. When a client connects to mogilefs, it established a connection with the **parent** process. This means idle connections do not utilize or hold individual query worker processes.

When a request is issued, the parent process sends the request to an idle worker for processing. If there are no idle workers available, the request is queued for later processing. You can monitor pending queries, and the average number of queryworkers in use via the `!stats` command.

Having too many query workers will add unnecessary load to your tracker and database. Workers have short caches for some common data fetched from the database.

## The Job Master ##

The job\_master process is a special worker. There is at most only **one** per tracker instance. The job master consolidates fetching jobs to process from database queues. It monitors and populates the internal work queues for delete, replicate, fsck, etc. It is similar to a Gearman job server with a persistent and shared queue, but more lightweight. This was recently introduced in 2.30 and helps greatly increase the scalability of a tracker. Since individual workers no longer have to poll the database, adding more doesn't cause idle load.

## Delete Workers ##

Process delete queues. Permanently removes deleted or overwritten files, and removes dead tempfiles from failed uploads. If your `file_to_delete2` table is expanding too much, try adding more.

## Reaper Workers ##

Reapers quickly process devices which have been recently marked as 'Dead'. They delete the location of copies files might have on the dead devices, and schedule replication to attempt to repair the files. The worker is very aggressive to facilitate a speedy recovery from the loss of a device.

## Monitor Worker ##

The monitor worker is another unique-per-tracker process. The monitor constantly contacts all devices and checks the database to see which ones are available and online. If a device becomes unavailable on its own, it's the job of the monitor process to note that change to all the workers a tracker runs, and to note its return if it does.

## Replication workers ##

Replication workers are the meat of MogileFS's magic file management. It checks a file against a replication policy, and will make more copies in specific places based on what the policy dictates. It also handles Drain and Rebalance. If you notice it taking too long to replicate files, add a few more workers.

TODO: Add a seperate page for tuning and replication options.

# Tuning Worker Counts #

Jobs start with a number of workers as specified in the /etc/mogilefs/mogilefsd.conf file. They can be monitored and adjusted at runtime via the `!jobs` and `!want` commands.

Unfortunately these commands aren't easily maintainable via the mogadm command, so we access trackers individually to tune
```
$ telnet localhost 7001
Trying 127.0.0.1...
Connected to localhost.localdomain (127.0.0.1).
Escape character is '^]'.
```

```
!jobs 
delete count 1
delete desired 1
delete pids 3736
fsck count 1
fsck desired 1
fsck pids 9012
job_master count 1
job_master desired 1
job_master pids 3769
monitor count 1
monitor desired 1
monitor pids 3767
queryworker count 30
queryworker desired 30
queryworker pids 3737 3738 3739 3740 3741 3742 3743 3744 3745 3746 3747 3748 3749 3750 3751 3752 3753 3754 3755 3756 3757 3758 3759 3760 3761 3762 3763 3764 3765 23113
reaper count 1
reaper desired 1
reaper pids 3768
replicate count 4
replicate desired 4
replicate pids 32669 32670 32671 32672
```

Adjusting jobs is simple.

```
!want 5 fsck
Now desiring 5 children doing 'fsck'.
.
!want 1 fsck
Now desiring 1 children doing 'fsck'.
.
```

Requesting fewer workers will have the parent process slowly kill them off (gracefully, as they finish work). Requesting more will have the parent spawn more workers. If you wish to stop or reset all of your workers, `!want 0 replicate` is a good approach. Wait until they're all gone, then start them up again.