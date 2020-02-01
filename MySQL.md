[Back To Maintenance Page](Maintenance.md)

# Tips for MySQL and MogileFS

Most MogileFS instances use MySQL as the metadata store. It's generally a good idea to have more than one of these. Multiple instances can be used for HA (Highly Available) pairs, read slaves for scalability, etc.

-------------

# MySQL and HA #

If you're wondering about setting up highly available MySQL databases, the best thing you can do for yourself is hire a contractor (percona, openquery, etc), or pick up some books and start reading. There is a **huge** amount of information available related to managing multiple MySQL instances and MySQL replication. Most of it is beyond the scope of this document. We will talk about how it can specifically apply to MogileFS however.

## MogileFS's Requirements ##

MogileFS is a distributed data store, which can have many storage nodes and many trackers. However, it has to have a **single** metadata store, which is the point where the trackers all coordinate. I'll repeat since it's worth making a point here: **ALL** trackers **MUST** point to the **SAME** database instance. They use advisory locking to ensure they don't collide in replication, and coordinate queue processing via transactions. Without this you may **permanently lose data**.

Two trackers will talk to the same database, and use locking features to ensure they both don't operate on the same file at the same time by accident, among many other things.

This doesn't mean you cannot use an HA MySQL instance with mogilefs however, not at all. So long as you ensure your mechanism points all trackers at the same DB, it will work fine. They can even tolerate a short outage.

To reiterate again: Master/Master replicated setups work **great** with MogileFS, but they must be configured Active/Passive.

## Examples of HA ##

VIPs (virtual IP's) and proxies are usually the way to go. Simpler options exist if you can't do either, even.

### Automatic VIP ###

The most common setup is simply using a "floating" third IP address between two Master/Master replicated databases. This can be managed by Heartbeat from the Linux-HA project, or any other utility. The writer uses a somewhat modified utility called [Flipper](http://provenscaling.com/software/flipper/) to manually move a flip around after a DB has died or needs repair.

### Manual VIP ###

You can use flipper, or really anything. interface scripts and a wrapped arp command are all you need to manage a VIP between two hosts. Some schools of thought say automatic failover between DBs is okay, but a lot of seasoned DBA's prefer manual failover. If the DB is in a weird position or had broken replication, it can be better to leave the system broken for a few minutes to ensure userdata desn't enter a time warp.

### Proxy ###

Many TCP proxies and load balancers can be used to emulate VIPs. Trackers talk to the proxy, and the proxy points all connections to the database. If the proxy detects connection failures against the first server, it can cut over to the secondary. I'm not providing links since this is an overview of what's possible and what works.

Be **very** careful to not accidentally load balance against the databases. If your TCP proxy is sending connections from the VIP down to both databases, you end up with a broken setup.

## Flipping Sides ##

If you're moving a VIP or proxy from master to slave, the key is to do it fast, and make sure replication is caught up at the time of the flip. Tools like flipper manage this for you, so be wary of any automatic failover setups you might use.

-------------

# Database Backups #

You really ought to be familiar with any of the multitude of documentation available to MySQL about backups. This writer finds that maintaining Active/Passive Master/Master pairs is a livesaver. netcat and tar are very quick ways to back up a momentarily stopped database. They also restore very quickly. Then you just have to patch binlogs.

It's important to remember to take backups of the metadata DB. Who knows what hardware failure, software bugs, or operator error have in store for you.

-------------

# MogileFS and MySQL Replication Slaves #

MogileFS can be configured to use replicated read slaves to spread read query load. Note that as of this writing, it "only" moves queries for the file and file\_on table to slaves. It should also be noted that caching paths outside of mogilefs (in memcached or elsewhere) will give you a much higher overall performance than adding read slaves and trackers.

Slaves are configured via mogadm
```
$ mogadm slave
Help for 'slave' command:
 (enter any command prefix, leaving off options, for further help)

  mogadm slave add <slave_key> [opts]                Add a slave node for store usage
  mogadm slave delete <slave_key>                    Delete a slave node for store usage
  mogadm slave list                                  List current store slave nodes.
  mogadm slave modify <slave_key> [opts]             Modify a slave node for store usage
```

You can discover what [opts](opts.md) are by specifying most of the command:
```
$ mogadm slave add

ERROR: Missing argument 'slave_key'

Help for 'slave-add' command:

  mogadm slave add <slave_key> [opts]                Add a slave node for store usage

      --dsn=s              DBI DSN specifying what database to connect to.
      --password=s         DBI password for connecting.
      --username=s         DBI username for connecting.
```

A `--dsn` might look like `--dsn "DBI:mysql:mogiledb:host=mogiledbhost"`. --username and --password should be obvious.

Be careful about getting the DSN wrong. Your tracker might act funny until the slave is removed again.

Finally, it's worth noting that the slave code is relatively young. As of writing, there are some old posted patches to improve it and some further ideas. However at present it doesn't take slave replication lag into account, or have great error handling.
