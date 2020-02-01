# Server Settings Overview

MogileFS supports server settings shared across all trackers.  These
server settings are stored in the database and not to be confused with
the per-tracker configuration files.

Server settings are accessible via the `mogadm settings` subcommand.

## Optimization settings ##

### skip\_mkcol ###

```
* skip_mkcol=(0|1) default: 0 (false, trackers attempt MKCOL)
```

Ensures trackers will skip making MKCOL requests entirely.  Enable this if your
entire storage cluster is capable of creating new directories on PUT requests.

nginx needs the "create\_full\_put\_path on;" option to take take advantage
of this setting.

Both mogstored (Perlbal) and cmogstored (v0.2.0+) create directories
automatically with PUT requests.

For DAV servers which do not support MKCOL at all (e.g. mogstored and
cmogstored), each tracker process will remember MKCOL failures
after the first failed requests.

Available since 2.58

### skip\_devcount ###

```
* skip_devcount=(0|1) default: 0 (false, devcount column is updated)
```

The devcount column is only useful for some statistics but not useful
internally to MogileFS.  Skipping updates to it will reduce I/O load
on the database.

If you've disabled devcounts and later want to restore them, setting
this back to zero (false) and running fsck (from the beginning) will
restore devcounts:

```
	mogadm settings set skip_devcount 0
	mogadm fsck reset
	mogadm fsck start
```

Available since 2.58

## Compatibility settings ##

### case\_sensitive\_list\_keys ###
```
* case_sensitive_list_keys=(0|1) default: 0 (database dependent)
```

This makes the the "after" and "prefix" parameters of the "list\_keys" command
behave case-sensitively.

The database dependent defaults are as described below:

MySQL is dependent on the charset of the "file" table, this is usually
case-insensitive by default (consult with your DBA).

Postgres is case-sensitive by default.  Postgres deployments are completely
unaffected by this setting.

SQLite is case-insensitive by default.  This server setting is the only
way to enable case-sensitive "list\_keys" behavior in MogileFS

Available since 2.64

## Network Zone settings ##

[Examples for multiple network support](ConfigureMultiNet.md)

### network\_zones ###
```
* network_zones=comma-delimited list of zone names
```

See [ConfigureMultiNet](ConfigureMultiNet.md)

### zone_${ZONE\_NAME} ###_

```
* zone_${ZONE_NAME}=netmask
```

See [ConfigureMultiNet](ConfigureMultiNet.md)


## Queue/FSCK settings ##

### queue\_size\_for\_fsck ###
```
* queue_size_for_fsck=INTEGER (default: 20000)
```
See [FSCK](FSCK.md) for more information.  Available since 2.33

### queue\_rate\_for\_fsck ###
```
* queue_rate_for_fsck=INTEGER (default: 1000)
```
See [FSCK](FSCK.md) for more information.  Available since 2.30

### internal\_queue\_limit ###
```
* internal_queue_limit=INTEGER (default: 500)
```

This controls the sizes of the internal queue for "fsck", "delete", "replicate",
and "rebalance".  Each task has its own internal queue, but this limit governs
all of them.

See [FSCK](FSCK.md) for more information.  Available since 2.33

### queue\_size\_for\_rebal ###
```
* queue_size_for_rebal=INTEGER (default: 10000)
```

Controls the number of rows in the "file\_to\_queue" database table for rebalance.

Available since 2.40

### queue\_rate\_for\_rebal ###
```
* queue_rate_for_rebal=INTEGER (default: 500)
```

Controls the number of FIDs to inject into "file\_to\_queue" per second for
rebalance.

Available since 2.40

### fsck\_checksum ###
```
* fsck_checksum=(MD5|off|class) (default: "class")
```

See [Checksums](Checksums.md) wiki page for more information.  Available since 2.60

## Cache settings ##

Read "doc/memcache-support.txt" in the MogileFS-Server source tree
before deciding to enable caching:

https://raw.github.com/mogilefs/MogileFS-Server/master/doc/memcache-support.txt

### memcache\_servers ###
```
* memcache_servers=comma-delimited list of $HOST:$PORT (default: "", none)
```

### memcache\_ttl ###
```
* memcache_ttl=INTEGER (default: 3600)
```