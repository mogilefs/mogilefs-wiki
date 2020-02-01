# What is MogileFS?
If you're here, chances are you know what MogileFS is.  If not, it's a distributed storage mechanism similar to Amazon's S3, except you own and control it. Typically it runs on linux, uses standard databases, e.g., MySQL and any underlying filesystem you wish.

More formally, MogileFS is our open source distributed filesystem. Its properties and features include:

  * Application level -- no special kernel modules required.
  * No single point of failure -- all three components of a MogileFS setup (storage nodes, trackers, and the tracker's database(s)) can be run on multiple machines, so there's no single point of failure. (you can run trackers on the same machines as storage nodes, too, so you don't need 4 machines...) A minimum of 2 machines is recommended.
  * Automatic file replication -- files, based on their "class", are automatically replicated between enough different storage nodes as to satisfy the minimum replica count as requested by their class. For instance, for a photo hosting site you can make original JPEGs have a minimum replica count of 3, but thumbnails and scaled versions only have a replica count of 1 or 2. If you lose the only copy of a thumbnail, the application can just rebuild it. In this way, MogileFS (without RAID) can save money on disks that would otherwise be storing multiple copies of data unnecessarily.
  * "Better than RAID" -- in a non-SAN RAID setup, the disks are redundant, but the host isn't. If you lose the entire machine, the files are inaccessible. MogileFS replicates the files between devices which are on different hosts, so files are always available.
  * Flat Namespace -- Files are identified by named keys in a flat, global namespace. You can create as many namespaces as you'd like, so multiple applications with potentially conflicting keys can run on the same MogileFS installation.
  * Shared-Nothing -- MogileFS doesn't depend on a pricey SAN with shared disks. Every machine maintains its own local disks.
  * No RAID required -- Local disks on MogileFS storage nodes can be in a RAID, or not. It's cheaper not to, as RAID doesn't buy you any safety that MogileFS doesn't already provide.
  * Local filesystem agnostic -- Local disks on MogileFS storage nodes can be formatted with your filesystem of choice (ext3, XFS, etc..). MogileFS does its own internal directory hashing so it doesn't hit filesystem limits such as "max files per directory" or "max directories per directory". Use what you're comfortable with.
  
---------

# High level overview

The following is a high-level overview of how MogileFS works, and what all the moving pieces are.

## Components
The involved parties are: 

  * **Application**: thing that wants to store/load files
  * **Tracker** (the mogilefsd process): event-based parent process/message bus that manages all client communication from applications (requesting operations to be performed), including load balancing those requests onto "query workers", and handles all communication between mogilefsd child processes. You should run 2 trackers on different hosts, for HA, or more for load balancing (should you need more than 2). The child processes under mogilefsd include:
    * Replication -- replicating files around
    * Deletion -- deletes from the namespace are immediate; deletes from filesystems are async
    * Query -- answering requests from clients
    * Reaper -- reenqueuing files for replication after a disk fails
    * Monitor -- monitor health and status of hosts and devices
    * ...
  * **Database** -- the database that stores the MogileFS metadata (the namespace, and which files are where). This should be setup in a HA config so you don't have a single point of failure.
  * **Storage** **Nodes** -- where files are stored. The storage nodes are just HTTP servers that do DELETE, PUT, etc. Any WebDAV server is fine, but mogstored is recommended. mogilefsd can be configured to use two servers on different ports... mogstored for all DAV operations (and sideband monitoring), and your fast/light HTTP server of choice for GET operations. Typically people have one fat SATA disk per mountpoint, each mounted at /var/mogdata/devNN.

## High-level flow

  * app requests to open a file (does RPC via library to a tracker, finding whichever one is up). does a "create\_open" request.
  * tracker makes some load balancing decisions about where it could go, and gives app a few possible locations
  * app writes to one of the locations (if it fails writing to one midway, it can retry and write elsewhere).
  * app (client) tells tracker where it wrote to in the "create\_close" API.
  * tracker then links that name into the domain's namespace (via the database)
  * tracker, in the background, starts replicating that file around until it's in compliance with that file class's replication policy
  * later, app issues a "get\_paths" request for that domain+key (key == "filename"), and tracker replies (after consulting database/memcache/etc), all the URLs that the file is available at, weighted based on I/O utilization at each location.
  * app then tries the URLs in order. (although the tracker's continually monitoring all hosts/devices, so won't return dead stuff, and by default will double-check the existence of the 1st item in the returned list, unless you ask it not to...)