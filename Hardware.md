# MogileFS Hardware Usage #

## The Goal ##

The design MogileFS follows is flexibility, commodity, inexpensive hardware requirements (this doesn't mean cheap/crappy). Avoiding vendor lock-in, and allowing you to mix, match, and choose what works best for your application. It is designed to not require assembling RAID volumes on storage nodes, in fact we highly recommend against doing that. Individual/smaller drives can be cheaper, and will scale writes better. Replication is usually between distinct hosts, so multiple copies on a single host is redundant. RAID or otherwise.

## Shared Hosts ##

MogileFS components can run on shared hosts, if you have free capacity. Trackers will use some CPU and some memory, storage nodes can be ran on CPU (web/etc) nodes if you add a few harddrives to them. The database can be shared with a larger database. As your installation grows, moving out to dedicated hardware isn't too hard.

## Dedicated Hosts ##

Notes and recommendations for having hardware dedicated to MogileFS components.

### Trackers ###

Trackers are easy to satisfy. They will use as much memory as you run worker processes. It will vary based on your version of perl, your host OS, and what version of MogileFS you run. Much of the memory (code/etc) will be shared between processes as well. Trackers access the database heavily, memcached if you have it, and can use a lot of network traffic with replication. Files are copied from a source node to a destination node with the tracker as a proxy, so traffic is increased.

Trackers cause no disk activity (aside from local syslogs, if you send those to disk). So wimpy/non redundant disks are fine. Trackers don't need to be on heavily redundant hardware, since clients should support trying multiple trackers if accessing one fails.

CPU usage can be heavy with many fsck/replicate workers running on a large set of jobs. This can be easily capped by limiting the number of workers running and allowing large batch jobs to take longer to process.

### Database ###

The trackers are generally light against the database. In versions 2.30 and later most of the remaining DB-heavy processes are being optimised. If you properly cache files and/or paths outside of mogilefs, get requests should be light and DB activity little. It can spike with a large amount of replicate/fsck/etc workers running at the same time, or heavy uploads, etc.

It's highly recommended to use master/master databases. MySQL is the most heavily optimised, but Postgres is well supported as well. **Please notes**, that regardless of how you replicate your databases, you must **only**, ever, point mogilefs trackers at a **single database**. Pointing them at different halves of a master/master cluster will break replication and possibly cause failures in file uploads. DB scaleout has not been necessary (on a properly configured setup, disk space is often a problem before DB capacity), but is coming properly in the future. For the moment mogile must point all trackers at the same database. If you're using MySQL, InnoDB is a requirement for reliability and speed. MyISAM will not scale so well and can lose data.

Hardware recommendations for a DB will vary. Small setups can use small machines with [R1](https://code.google.com/p/mogilefs/source/detail?r=1) disks. Larger databases should consult DBA's, or performance websites for up to date recommendations. The author of this article generally goes with fast SAS drives, a RAID card with a battery backed write cache (crucial!), and plenty of RAM. Flash drives are slowly becoming economical, but may not be necessary given MogileFS's requirements.

### Storage ###

Storage nodes are where the greatest tradeoff decisions happen. Generically: do you want more nodes with fewer drives, or fewer nodes with many large drives? Or few nodes with large drives? Do you use hot swap drives or cheaper, non hot swappable drives?

Storage nodes will generally not use a lot of CPU. Mogstored is used for monitoring and (by default) file uploads. Large setups typically use apache or nginx to manage read load. All of the above will likely not tax your CPU before running you out of IO capacity.

The tradeoffs that are important to remember are:

  * More nodes gives you the ability to (more cheaply) have a higher RAM/disk ratio. The more RAM is on the backends, the more files sit in the OS file cache, the faster it performs.
  * When a node fails completely, you must mark all of its devices as dead (or move them to a new chassis with the same IP/device names/etc). It will take time for MogileFS to re-replicate files which were on the dead host.
  * Versions 2.30+ reduce node recovery time significantly, but a very large node with many large disks can possibly take days to recover from losing.
  * You need to ensure you have enough IO capacity to satisfy your app/frontend requirements. Larger drives can store more files, but have a hard limit in IO capacity. More smaller drives means more IO's. Don't buy a sack of 2TB+ drives and discover they're far too slow.
  * Caching files outside of the system (Varnish, Squid, CDN) will affect how much IO and tracker capacity are used.
  * Whether or not storage nodes use expensive RAID cards with hot swap features is a matter of taste than a requirement. MogileFS is designed to **not** require RAID'ing devices you add. So in effect you probably don't need expensive cards at all.

A large system could have anywhere from 4 to 48 drives in a single machine. Adding storage nodes to MogileFS is extremely cheap and does not add notable load onto trackers. There's no communication between nodes (aside from required replication, and individual file requests), so adding hardware can only add capacity, as far as your network will allow.

Hot swap or not is a matter of preference/taste. See the section below on "dead hardware".

### Frontend/Proxy ###

The frontend is typically perlbal, or some concoction of an application/nginx/etc. It's highly recommended that path lookups from trackers are cached in your application, via memcached or similar. Paths can be cached for a few minutes or a few hours. Supply more paths to your cache, and perlbal/nginx will skip over ones that may be inoperable.

Sticking varnish or a CDN up front to cache frequently accessed files can help further reduce the strain on backend nodes and trackers. This is, again, a matter of taste. Some folks will prefer to have beefier storage nodes and reproxy to the backend every time. Others will prefer to have beefier caching nodes up front to serve common files in more controlled ways.

## What To Do With Dead Hardware ##

If you have a drive failure, what should you do? Should you spring extra for dedicated RAID controllers with hot swap backplanes, or can you go cheaper and use fast SATA/SAS cards and directly attach drives?

If you're OCD about having every drive bay active, go ahead and go the hot swap route. The cases are more expensive, backplanes, cards, etc. However you can easily slide out and replace a dead drive. When you do so, simply mark the old device dead, insert a new drive, and give it a **new device id**. Do not reuse the old one!

The other end of the spectrum is to use cold swappable hardware, with drives plugged directly into cheaper (but still fast!) SATA cards. When you order hardware, buy a few more drives than you actually need for the next year. Lets say 5%-10% more or so. When a drive fails in a device, simply mark it as 'dead' and move on. The failure rate for drives may well be low enough that it'll be necessary to replace the entire host before more than a few drives have failed in each. Or, if you're expanding rapidly, you're adding new hosts anyway.

The few extra drives ordered should work out to save costs over getting $700 RAID cards and extra expensive cases. Not doing work to swap drives in a datacenter can reduce manpower costs as well.

If you have many hosts, you could simply afford to power down dead hardware and ignore it until the end of the week/month/quarter/etc.

In the end, it's a matter of taste.
