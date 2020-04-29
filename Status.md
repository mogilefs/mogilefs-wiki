# Tracker status 

Newbies may notice that there is no any way to monitor system status by logs. 
Instead, you can checkout tracker status via the tracker admin (not mogadm) command.


## Howto

```
telnet localhost 6001
Trying ::1...
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
!stats
uptime 1615930
pending_queries 0
processing_queries 3
bored_queryworkers 37
queries 41374704
times_out_of_qworkers 8041
work_queue_for_delete 150
work_queue_for_fsck 200
work_queue_for_replicate 650
work_sent_to_fsck 380800
work_sent_to_replicate 14657511
```

As you can see above, you can check uptime, queue length, number of processing queries and so on.

It's worthy to mention that `times_out_of_qworkers` means number of times that a new query is received but all query works are busy doing other things. 
The new query is enqueued instead of processed immediately. You can evalute this metric with `pending_queries` and `bored_queryworkers` to determine whether to incease number of query works or to have more tracker instance.

## Protips

### Metrics are local

Those metrics are not global. If you have several trackers in one cluster, you may observe that metrics are vary cross each instance.

### You can use 3rd party tools to collect historal metrics

Metrics exposed by tracker represent only realtime status. 
If you want to see trends (That's really helpful while diagnosising system problems), Try to use [3rd party tools](Addons.md) to collect numbers periodicly.

