# Common MogileFS Maintenance

* Running a Filesystem Check - See the [FSCK](FSCK.md) guide.
* Rebalancing Files - See the [Rebalance](Rebalance.md) guide.
* Managing Tracker Jobs - See the [Jobs](Jobs.md) guide.

---

# Managing Hosts #

## Adding Hosts ##

Adding hosts is straightforward via `mogadm host add`

```
$ mogadm host add mystorage --ip=10.0.0.1
```

Note that it is highly recommended (and will become required) that you specify
the IP address of the host when adding it. The IP address mogilefs uses is
what your webservers and trackers will use to talk to the storage nodes.

It's common for people to alias "mystorage" to "127.0.0.1" via a hosts file,
which will cause mogilefs to issue paths for "127.0.0.1", unless you specify
an IP address.

**NOTE** When adding new hosts to a cluster, it could take a minute or two for
them to be recognized as caches expire.

## Taking Hosts Down Temporarily ##

If you need to do maintenance on a host (upgrade RAM, replace OS, etc) that
involves shutting it off, it's recommended you first mark it as "down". While
MogileFS is built to be resilient to accidental failures, there's nothing
wrong in giving it a heads up before taking a host offline.

```
$ mogadm host mark mystorage down
... do work....
$ mogadm host mark mystorage alive
```

## Taking Hosts Down Permanently ##

If you mark a host as 'dead', it is not recommended to ever mark it as 'alive'
or 'down' again. You must only ever mark things as "dead" which you don't
expect to ever bring back to life again.

If you mark a host as 'dead', mogilefs will **not** re-replicate files on that host. You **must** mark the individual drives as 'dead'.

```
$ mogadm host mark mystorage dead
```


---

# Managing Devices #

## Numbering Devices ##

New devices should **always** have incremental devids. When you add new disks, give them new, unique numbers.

If you replace a dead disk with a new disk, **always** give it a new devid. **never** reuse the old devid. This is so you can replace a disk in mogilefs before all of the files from the old device have finished re-replicating. Otherwise you could end up clobbering fids uploaded to the replaced device, or never re-replicate old files. Bad! Don't do it!

## Adding Devices ##

```
$ mogadm device add mystorage 5 --status=alive
... or...
$ mogadm device add mystorage 5 --status=down
```

The above shows adding a storage device (#5) to storage host (mystorage)
either defaulting to alive or down. Good practice is to add a new set of
devices as "down", then quickly review everything works and allow caches to
pull in the new devices. Then mark them all as alive when you're ready for
them to be used.

It's important that you not run the storage nodes as `root` if you can avoid
it, and that the underlying directories below where the devices are stored are
owned by root and not the mogstored user. If you unmount a device, MogileFS
will happily write to the underlying filesystem if it still can. There's no
way for it to know if the directory is a mount point or not.

## Device Maintenance ##

MogileFS does some automatic testing of devices and hosts. Each tracker has a
`monitor` job which will periodically probe devices and ensure it can, write,
read, and delete small files on the device. In the case of a bad filesystem
failure causing a filesystem to be remounted read-only, or disappearing
entirely, mogile should stop trying to write new files into the device.

In the case of a temporary downtime, such as taking time to unmount the
filesystem and run the OS `fsck` on it, you can mark the device as `down` and
then later `alive` again. In the case of filesystem corruption, IO errors,
etc. It's safest to immediately mark the device as `dead` and allow MogileFS
to make new copies from still-good devices.

### Temporary Maintenance ###

```
$ mogadm device mark mystorage 5 down
... later...
$ mogadm device mark mystorage 5 alive
```

Please avoid leaving devices or hosts marked as `down` for long periods of
time. MogileFS might queue up deletes, replications, fsck checks, etc, waiting
for the device to come back up. If a device is going away forever, you must
mark it as `dead`. This doesn't mean there's a set time limit in minutes or
hours. That all depends on how fast things change in your setup and how big
you want your queues to be.

### Readonly and Drain modes ###

If you want to freeze all files on a device the way they are, you would use
`readonly` mode. This will stop MogileFS from putting new files on the device,
but it will **also** prevent **deletes** from removing files. Instead the deletes
will queue up and wait for you to mark the device as `alive` or `drain`.

```
$ mogadm device mark mystorage 5 readonly
$ mogadm device mark mystorage 5 drain
```

`drain` mode, as of version 2.40 and higher, tells MogileFS that no new files
should be written to a device. However files may be deleted while in drain
mode.

So if you have devices where you do not wish to put new files, mark them all
as `drain`.

**NOTE** In earlier versions of MogileFS, `drain` mode would actively remove
fids from a device. It has since been replaced by [Rebalance](Rebalance.md).

## Re-Replicating Files ##

If a drive dies, mogilefs will attempt to avoid the device, but it will not automatically re-replicate files which were on the device. You must manually mark it as 'dead' via mogadm. As soon as you do so, mogilefs will start to remove files that were on the device and attempt to re-replicate them.


---

# Managing Databases #

MogileFS **must** have **all** trackers point to the **same database** at **all times**. There are no exceptions! Transactions and exclusive locks are used to coordinate individual processes and trackers, so running trackers against multiple, master-master replicated databases, will cause sporadic failures.

See [MySQL](MySQL.md) for more detailed information specific to MySQL.