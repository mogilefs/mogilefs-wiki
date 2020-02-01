# How To Setup MogileFS #

## Overview ##

This document explains how to set up and configure a basic MogileFS installation. It is intended for the novice systems administrator and should enable anybody with the skills to install a basic Linux distro to get MogileFS up and running on it.

If you are new to MogileFS, we recommand you read  [Getting Started Guide](QuickStartGuide.md) first.


I am going to assume you have three roles for this setup. A machine should be able to handle more than one role. I will use hosts `mogiledb.yourdomain.com`, `mogiletracker.yourdomain.com` and `mogilestorage.yourdomain.com` – rename these to meet your needs.

## Getting MySQL Libraries ##

Mogile requires a SQL database. These instructions presume that you are installing MySQL. You'll need the client libraries installed even if your MySQL server is running on a different machine.  On Ubuntu or Debian, the easiest install is using apt-get:

```
% sudo apt-get install libdbd-mysql-perl
```

## Getting MogileFS ##

Perl comes with CPAN, and MogileFS is in CPAN, along with its dependencies. Easiest way to get it is to fetch it via CPAN like so:

```
% sudo perl -MCPAN -e 'install "MogileFS:Server"'
```

It will ask you before it installs dependencies. Just hit ENTER for every question it asks.

Note that the server actually requires the client software (because it uses the API library), so that will be installed automatically. You will almost certainly want the Utility programs as well, so fetch them like so:

```
% sudo perl -MCPAN -e 'install "MogileFS:Utils"'
```

It's highly advisable that you use a released version from CPAN. However, if
bug hunting or interested, the latest code is on github:
http://github.com/mogilefs/

[Here](InstallFromCPAN.md) are some more details about installing mogilefs from CPAN, for reference.

### Installing Perl Module Dependencies ###

As for Perl modules, you need (at least) Perlbal and Danga-Socket and any
dependencies. If you are not sure how to locate the dependent modules,
consider using CPAN (http://cpan.org/) as it can install a module and all of
the dependencies. If you already have a reasonable Perl installation you
probably have most of the non-MogileFS modules already.

IO::AIO is also required for mogstored, and Perlbal::XS::HTTPHeaders is a
highly recommended optional dependency for Perlbal.

**NOTE** Some versions of IO::AIO (between 3.261 and 3.65) contain a bug which
may prevent mogstored from working. You must either upgrade IO::AIO to 3.65+ or
upgrade Perlbal to 1.76+.

## Installation ##

### Creating a Database ###

Setting up a MySQL server is beyond the scope of this document – there are packages available for most distributions, find one that suits your needs. Once you have a MySQL server up and running on host mogiledb.yourdomain.com, create a table and user for MogileFS. Some libraries don't play well with new MySQL passwords – if you use these, set the password using the “OLD\_PASSWORD” function. Make sure you change the password to something better than the example.
```
# mysql
mysql> CREATE DATABASE mogilefs;
mysql> GRANT ALL ON mogilefs.* TO 'mogile'@'%';
mysql> SET PASSWORD FOR 'mogile'@'%' = OLD_PASSWORD( 'sekrit' );
mysql> FLUSH PRIVILEGES;
mysql> quit
```
You will also need to create the schema – that is covered later in this document.

Note that MogileFS also works with Postgres, however the setup is not documented here.

### Setting up the Trackers and Storage Servers ###

If you do wish to run the tests, you'll need a database. You must first export some environment variables, if your test database isn't in the default location:
```
export MOGTEST_DBHOST="192.168.1.50"
export MOGTEST_DBPORT="3306"
export MOGTEST_DBUSER="mogtest"
export MOGTEST_DBPASS="mogtest"
export MOGTEST_DBNAME="mogtest"
export MOGTEST_DBTYPE="MySQL"
```

## Setup ##

### Database Configuration ###

The database is empty and will need a schema applied. The server code has a utility named 'mogdbsetup' to make this process simple. By default it assumes the database is located on localhost:
```
# ./mogdbsetup --dbname=mogilefs --dbuser=mogile --dbpassword=sekrit
```
but if you are running it from a different host you will need to provide the host name on the command line:
```
# ./mogdbsetup --dbhost=mogiledb.yourdomain.com --dbname=mogilefs --dbuser=mogile --dbpassword=sekrit
```

Again, make sure you replace the host and password so that they match your database configuration from above.

The mogdbsetup utility does not specify a table type by default, so your tables will match the defaults for your database. In many cases this will mean that you end up with MyISAM tables. If you prefer InnoDB tables you will either need to make sure your database defaults to InnoDB, or you can manually convert the tables (both of these are outside of the scope of this document but there are plenty of examples out there).

### Tracker Configuration ###

On each tracker server (mogiletracker.yourdomain.com), create a configuration file at /etc/mogilefs/mogilefsd.conf with the following:
```
db_dsn = DBI:mysql:mogilefs:host=mogiledb.yourdomain.com;port=3306;mysql_connect_timeout=5
db_user = mogile
db_pass = sekrit
conf_port = 7001
listener_jobs = 5
node_timeout = 5
rebalance_ignore_missing = 1
```

db\_dsn points to your database instance. If you are running the database on the same machine as the storage server, you can omit ":host=mogiledb.yourdomain.com: and it will use the local machine. db\_user and db\_pass should match the user and password you configured when setting up your database.

The program 'mogilefsd' will not run as root, so you will need to run this as a non-root user. To create a user for this, enter the following command and follow the prompts to create the "mogile" user:
```
# adduser mogile
```

In order to use the tools to set up the storage servers, you will need to have the trackers running. Refer to "Starting Trackers", below.

If you want the tracker to use memcached:
```
# mogadm --lib=/usr/local/share/perl/5.8.4 --trackers=192.168.42.1:7001 settings set memcache_servers 127.0.0.1:10000 
```

### Storage Server Configuration ###

On each storage server, create the storage directory (make sure it has access permissions for the user you will use to run mogstored):
```
# mkdir /var/mogdata
```

Configure it:

On each storage server, create a configuration file at /etc/mogilefs/mogstored.conf with the following:
```
httplisten=0.0.0.0:7500
mgmtlisten=0.0.0.0:7501
docroot=/var/mogdata
```

Use 'mogadm' to add each storage server to the database. This requires that the trackers are already running so if you have not already started them, refer to "Starting Trackers", below. You need to supply the Perl lib path which has the 'MogileFS.pm' perl module installed – this was installed if you installed the API in the "Setting up the Trackers and Storage Servers" section above. The following example would add the host mogilestorage.yourdomain.com as a storage server, assuming that mogilestorage.yourdomain.com had an IP address of 192.168.42.3 (listening on port 7500) and your tracker had an IP address of 192.168.42.1 (listening on port 7001):
```
# mogadm --lib=/usr/local/share/perl/5.8.4 --trackers=192.168.42.1:7001 host add mogilestorage --ip=192.168.42.3 --port=7500 --status=alive
```

You can confirm that your host(s) were added with the following command;
```
# mogadm --lib=/usr/local/share/perl/5.8.4 --trackers=192.168.42.1:7001 host list
```

You also need to add devices for each storage host. You will need to manually add a unique device id after the host:
```
# mogadm --lib=/usr/local/share/perl/5.8.4 --trackers=192.168.42.1:7001 device add mogilestorage 1
```

Finally, add the correctly-name device (folder) to each storage host. I have been unable to get the tools to handle this well, so I am probably doing something wrong. As a workaround, I used the modadm device list command to see what device names were assigned and then I added the folders to my storage hosts. Run the following command:
```
# mogadm --lib=/usr/local/share/perl/5.8.4 --trackers=192.168.42.1:7001 device list
```

It will list each host and the device name followed by its status and storage available. Here is example output:
```
mogilestorage 1: alive
used(G) free(G) total(G)
dev1: alive 0.892 67.772 68.664
```

This means "mogilestorage" has a host id of "1" and it has one device named "dev1" on it and that device is in the "alive" state (your other statistics will probably be zeros). Using the example output above, you would simply create the directory on mogilestorage.yourdomain.com:
```
# mkdir -p /var/mogdata/dev1
```

Finally, confirm that your devices are configured:
```
# mogadm --lib=/usr/local/share/perl/5.8.4 --trackers=192.168.42.1:7001 device list
```

To get IO stats working, you need to install 'iostat'. Example for debian/ubuntu
```
# apt-get install sysstat
```

## Running MogileFS ##

### Starting Storage Servers ###

Start each storage server (mogilestorage.yourdomain.com) by running the following command as root: (or your nonprivileged user)
```
# mogstored --daemon
```

### Starting Trackers ###

Trackers will not run as root, so you will need to run them as another user. If you created the "mogile" user when setting up the trackers, the following commands will work (assumes you start logged in to mogiletracker.yourdomain.com as root):
```
# su mogile
$ mogilefsd -c /etc/mogilefs/mogilefsd.conf --daemon
$ exit 
```

You can confirm that the trackers are running with the following command:
```
# ps aux | grep mogilefsd
```

If you don't get a list of running processes, the trackers are not running.

## Try It ##

### Do a Quick Sanity Test ###

The 'mogadm' tool can be used to make sure your trackers are functioning. You need to supply the Perl lib path which has the 'MogileFS.pm' perl module installed – this was installed if you installed the API in the "Setting up the Trackers and Storage Servers" section above. The following example would check all mogile components using the trackers at IP address 192.168.42.1 and 192.168.42.2, both listening on port 7001:
```
# mogadm --lib=/usr/local/share/perl/5.8.4 --trackers=192.168.42.1:7001,192.168.42.2:7001 check 
```

## Try it with Real Data ##

### Create a domain ###

```
# mogadm --lib=/usr/local/share/perl/5.8.4 --trackers=192.168.42.1:7001,192.168.42.2:7001 domain add testdomain
```

Add a class to the domain:
```
# mogadm --lib=/usr/local/share/perl/5.8.4 --trackers=192.168.42.1:7001,192.168.42.2:7001 class add testdomain testclass
```

## Next Steps ##

From here, move on to [CommandlineUsage](CommandlineUsage.md) to get a walk through of the demo
tools, or move straight to [AppExample](AppExample.md) to see an example of an application
using MogileFS.

## Troubleshooting ##

### When I run mogadm I get "Unable to retrieve host information from tracker(s)" ###

mogadm requires the tarckers to be running before it is run.

### When starting the storage daemon I get "ERROR: Directory not found for service mogstored" ###

You did not create a storage directory or you are starting the mogstored as a user that does not have access to the directory.

### Problems Connecting ###

Make sure your firewall is open. Using the examples in this document, port 7500 and 7501 needed on storage servers, 7001 on trackers.

### While testing I get “MogileFS::Backend: couldn't connect to mogilefsd backend at /usr/local/share/perl/5.8.4/MogileFS.pm line 56” ###

Make sure your tracker connects to the database:
```
# su mogile
$ mogilefsd -c /etc/mogilefs/mogilefsd.conf
```

### I get a "REQUEST FAILURE" on "Checking devices..." when doing a mogadm check ###

Confirm the devices (folders) exist in the /var/mogdata directory and that the use running the mogstored process has full permissions to these directories. If the device does not exist, add it – it will take a few seconds for mogadm check to reflect the fixed directory. For example, if mogilestorage.yourdomain.com had the device "dev1" on it, you would add the directory:
```
# mkdir -p /var/mogdata/dev1
```

Another possible source of this error is the HTTP\_PROXY environment variable set to a proxy, which for some reason fails to connect to the storage node(s). mogadm uses LWP library which honors this environment variable. To solve this error either unset the variable for mogadm to connect to the storage node(s) directly or fix the proxy problem.
