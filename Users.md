# MogileFS users

A list of some of the many current users of MogileFS.

## SixApart ##

(Note: SixApart is now SayMedia)

SixApart stores and serves all user uploaded images, videos, and music via MogileFS. It has been a long term scalable data store for files varying from a few kilobytes to many megabytes. SixApart typically uses larger storage nodes (10+ drives) on dedicated hardware.

Normally files are served directly from storage nodes to clients. Gallery thumbnails are generated on the fly and displayed to clients, then "cached" in mogilefs with a single path. Thumbnails which disappear are simply regenerated again.

## YellowBot ##

[YellowBot](http://www.yellowbot.com/) and [weblocal.ca](http://www.weblocal.ca/) since early 2009.  We replaced our previous storage in Amazon S3 to get more control over the namespaces, headers on delivery and to improve performance.  We're storing user and business-owner pictures, videos and PDFs.  We're also using MogileFS for storing application log files (which are then used for analytics in a mapreduce-ish fashion).

As per November 2009 it's about 2TB across 30 million files from a few KB to 1GB in size.   The files are split in two installations; each have two trackers and respectively 6 and 4 storage nodes.

The trackers and storage nodes are just whatever servers have some spare CPU or disks respectively. A small Perl web server parses requests and returns the response and reproxy headers to Perlbal.  The databases are in our regular replicated database setup; we use a CDN for end-user delivery so the load is very minimal.

## Wikispaces ##

[Wikispaces](http://www.wikispaces.com/) has been using MogileFS since February 2006.  The system stores all user-generated content -- wiki pages, images, files -- ranging from a few bytes to several GB per file.  As of November 2009, we have four trackers backed by a multi-master MySQL database, ~100M files, tens of TB of storage on both small (4 disk) and large (10 disk) storage nodes.  We use lighttpd to serve files from mogstored nodes and nginx reproxying.

## Dreamwidth Studios ##

[dreamwidth.org Dreamwidth Studios]

  * Using MogileFS since: April 6th, 2008 (when we initially setup the development environments)
  * 7 hosts 14 devices (~200G each)
  * ~340,000 files
  * 2 trackers
  * Quad core Q6600 2.4GHz 4GB RAM dual ~200GB drives (NO RAID)

The machines do other things -- we "stripe" our MogileFS installation across the cluster of web servers, job servers, etc...

## Mecanto.com ##

Company name: Hyrax Media.

Company site: http://mecanto.com

Using mogile since: Day one (early 2008). After some research this seemed like the fastest way to get what we needed and also cheap in the long run.

Cluster size: Don't want to supply details but we store well over 100TB on it.

Rough number of files stored: Over 5 million

Number of trackers: 1 (I know this seems very little), our system currently manages with this, although I'm sure we should add more.

## Footnote.com ##

[Footnote](http://www.footnote.com) has been using mogile since Sept 2008. We have 200TB of storage and 200+ million files. We run lots of 2U supermicro 12 drive servers as dedicated mogile storage servers including some on ZFS. We use nginx for all get/puts. All mogile boxes are dedicated machines. Mogile is great! The best part about mogile is the way the drive lights flicker due to the non-raid random access of the disks. It looks like the racks are thinking.

## Rentalia Holidays ##

[Rentalia Holidays](http://www.rentalia.com)

Using mogile since July 2009.

Rentalia.com is an independent company created by a team of tourism and Internet experts.  Our aim is to make the search for a holiday rental as easy as possible.

  * Cluster size (hosts, devices) 4 hosts,&nbsp; 4 devs
  * Rough number of files stored,&nbsp; 500k files
  * Number of trackers, 5 trackers
  * Regular hp prolliant,&nbsp; debian linux SO
  * We server image and invoice files using&nbsp; Valery Kholodkov's nginx+mogilfs
  * module

## KWICK! Community ##

We use MogileFS since the start of 2011 for our main image store at
KWICK! Community (http://www.kwick.de). We are one of the big Social
Networks in Germany and over 10 years on the market.

All profile pictures and photo galleries are stored in MogileFS.

Numbers:
  * Over 80 Mio of image files
  * 2 Trackers (2 x E5520 2.27GHz, 12 GB RAM)
  * 7 dedicated storage nodes (8 x 15k SAS HDDs)
  * 24 shared storage nodes (mostly 3-4 x 7k SATA HDDs)
  * 123 MogileFS devices, total of 64TB storage available
  * 4 load balanced Nginx MogileFS Frontend servers
  * 2 MySQL Servers with SSD RAID and 48 GB RAM (one is Hot-Standby)
  * PHP MogileFS Extension is used by the application
  * Storage nodes use nginx too
  * Images smaller than 1 MB are cached via srcache + memc nginx module
in memcache (dedicated storage nodes are also Memcache servers)

## Automattic ##

Company name: Automattic

Company website: WordPress.com, et al.

Using mogile since: Beginning of 2010

Cluster size (hosts, devices):
  * 3 separate production clusters (in 3 different datacenters)
  * Each cluster is 14 hosts, 45 devices per host for a total of 42 hosts and 1,890 devices

Rough number of files stored: About 625 million per cluster

Number of trackers: One per host, so 42 currently

Type of hardware:

Trackers/storage nodes:
  * Dell [R610](https://code.google.com/p/mogilefs/source/detail?r=610)/[R710](https://code.google.com/p/mogilefs/source/detail?r=710)
  * 2 x 5620 Xeon
  * 2 x 146GB RAID 1 local disks for the OS
  * 32GB RAM
  * 3 x MD1000 direct attached storage arrays per host
  * Each MD1000 has 45 x 1TB 7200 RPM SATA drives which we configure as JBOD

Database (x2 master/slave):
  * Dell [R610](https://code.google.com/p/mogilefs/source/detail?r=610)
  * 2 x 5620 Xeon
  * 48GB RAM
  * 6 x 146GB 15k SAS RAID 10

Other fun facts:

  * All hardware is dedicated to Mogile.
  * We use nginx + [mogilefs module](http://www.grid.net.ru/nginx/mogilefs.en.html)
  * We cache all files upstream of Mogile and use memcached-based path caching on the trackers
  * Adding about 6 nodes every 45 days to keep up with current growth
  * Hope to get to 1 billion files and beyond!

## Mood Media North America ##

We spent months outlining our dream storage system. After finalizing our decision we discovered Mogile. Mogile meet everyone of our design decisions (and more). We are thrilled that we didn't have to write it ourselves!

  * Company Name: Mood Media North America (Formly Trusonic)
  * Company website: http://us.moodmedia.com
  * Using mogile since 2008

  * Cluster size (7 hosts, 25 HDDs, 24TB total storage, 76% in use
  * Unique files stored: 3,264,057
  * Total files stored:  7,330,190
  * 3 dual purpose tracker/storage nodes, 4 storage only nodes.
  * Supermicro 1U `SuperServers` Intel Core2 2x2.45Ghz, 4 sata bays
  * Consumer grade Western Digital Hard drives 750GB to 2TB
  * Launched with 3 servers, added servers ~6months, no downtime.

## PBWorks ##

Company website: http://www.pbworks.com [www.pbworks.com]

Using mogile since: Beginning of 2009

Cluster size (hosts, devices):

2 production clusters
9 hosts, 12 devices per host, 216 TB/cluster

Number of trackers: One per host, 18 currently

Type of hardware:

Trackers/storage nodes:

SuperMicro SuperStorage 2U
1 x Quad-Core AMD Opteron(tm) Processor 2347 HE
1 x 2TB drive for core OS
16GB RAM
12 x 7200 RPM 2TB SATA-II drives per host as JBOD

Database (x2 master/slave):

SuperMicro 6017R-WRF
2 x Six-Core E5-2630
192GB RAM
LSI MegaRAID 9260-4I 4-Port 6Gb/s SATA/SAS PCI-E 8
2 x 15k 2TB SATA3 Disks in RAID 1

## GMO Pepabo, Inc. ##

Basic info:

  * Company name : GMO Pepabo, Inc.
  * Company website : https://pepabo.com/en/
  * Using mogile since : April 2008

Using scene: We has been using MogileFS in "30days Album"( https://30d.jp/ ) since April 2008.

  * Cluster size (hosts, devices)
    * 13 hosts
      * 3 x tracker nodes
      * 10 x storage nodes
    * 408 HDDs
    * 3.28 PB total storage
  * Rough number of files stored
    * Over 6 Billion of image files
  * Number of trackers
    * 3 trackers
  * Type of hardware
    * tracker nodes
      * DELL PowerEdge R430
      * 2 x Intel(R) Xeon(R) CPU E5-2620 v4 @ 2.10GHz
      * 4 x 16GB RAM
      * 1 x SAMSUNG SM863A 480GB
    * storage nodes
      * SuperMicro SuperStorage 6049P-E1CR36L
      * Intel(R) Xeon(R) Bronze 3104 CPU @ 1.70GHz
      * 2 x 16GB RAM
      * 1 x SATADOM 64GB

## Your company here ##

If you or your company use mogilefs and you can publicly share some information on this, please email the mailing list.

Basic info:

  * Company name
  * Company website
  * Using mogile since

Less basic, optional, and as of date of submission:

  * Cluster size (hosts, devices)
  * Rough number of files stored
  * Number of trackers
  * Type of hardware