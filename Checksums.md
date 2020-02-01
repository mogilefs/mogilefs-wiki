# Checksum mechanisms

MogileFS 2.60+ implements checksums support.  Checksums can be
stored in the database on a per-class basis, and/or files without known
checksums can be compared against other replicas during fsck.


## Per-class checksumming ##

Checksumming is available optionally on a per-class basis.  To enable
checksumming for a class, you can specify it's hashtype via mogadm:

> mogadm class modify --hashtype=MD5

Per-class checksumming uses the new "hashtype" column in the "class" database
table (in SCHEMA\_VERSION=15).


### Tracker protocol changes ###

"create\_close" gets two new, optional parameters

```
* checksumverify=(0|1) default: 0 (false)
* checksum=$HASHTYPE:$HEXDIGEST
```

If "checksumverify" is "1" and "checksum" is present, "create\_close"
will not return until it has verified the checksum.

If the storage class of a file specifies a valid "hashtype",
the checksum is saved to the "checksum" table in the database.

The client is _never_ required to supply a checksum, but supplying
one will speed up initial replication if a class requires it.

The client may always supply a checksum (including checksumverify)
even if the storage class of the file does not required.

Users of the Perl client library may specify these arguments using
the "create\_close\_args" parameter of the "new\_file" subroutine.


## Global fsck\_checksum server setting ##

For users unable or unwilling to store checksums in the database,
using the "fsck\_checksum" server setting allows you to checksum all
known copies to ensure all copies match.

If you want to checksum all files using MD5, regardless of per-class settings:

> mogadm settings set fsck\_checksum MD5

If you want a faster fsck without any checksumming, you may bypass
checksumming entirely:

> mogadm settings set fsck\_checksum off

The default is to rely on the per-class "hashtype" setting:

> mogadm settings set fsck\_checksum class


## Fsck error logging ##

See [FSCK](FSCK.md) for a full list of fsck error codes.

```
* BSUM - Database checksum didn't match on-disk contents

* NSUM - Database checksum was missing when a class required checksums

* MSUM - Only for fsck_checksum=$HASH users, multiple checksums were
         resolved and MogileFS did not know which is correct.
```

## Performance considerations with checksums ##

```
* Use MD5, it is inexpensive CPU-wise and it allows using the
  Content-MD5 HTTP header to verify PUT requests.

* Clients should supply a "checksum" parameter on "create_close"

* Use Perlbal 1.80+ and mogstored for handling PUT requests,
  it supports the Content-MD5 header so replicators (and clients)
  can skip expensive checksum verification.

* Do NOT use "checksumverify" if you're using MD5 and Perlbal 1.80+
  for PUTs.  Instead, generate the Content-MD5 header yourself
  and send that with a PUT.

* Use and configure the mogstored sidechannel port.  It avoids
  wasting network bandwidth when fsck needs to checksum on-disk
  contents.
```

If the above recommendations are followed, normal client
uploads/replication should suffer little performance degradation.
However, fsck with checksumming will inevitably take longer as all
replicas of all files must be reread off disk.