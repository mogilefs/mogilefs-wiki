# Deploying MogileFS manually for test #

Want to play with the MogileFS API? Want to help write code, test patches, fix
bugs? Follow the guide below to quickly build a test instance.

Assuming you have a Debian, Ubuntu, or Redhat-y system. OS X will be similar
but has some missing steps. For a more complete guide on how to set up an
install of MogileFS, refer to the [InstallHowTo](InstallHowTo.md)

No attempts to time this guide have actually been made; YMMV.

## Howto

Add a new local user and log into it
```
adduser mogilefs
# ...etc...
su - mogilefs
$
```

Use [App::cpanminus](http://search.cpan.org/dist/App-cpanminus/) to quickly
bootstrap locally installed perl modules.
(note: you might need to add --no-certificate-check if you have a buggy
version of wget)
```
wget -O- http://cpanmin.us | perl - local::lib
PERL5LIB="/home/`whoami`/perl5/lib/perl5" perl -Mlocal::lib >> ~/.bashrc
source ~/.bashrc
```

Install libdbd-mysql-perl (debian) or DBD::mysql via yum. This is needed even
if you're not using MySQL.
```
apt-get install libdbd-mysql-perl
... or ...
yum install perl(DBD::mysql)
```

Gas up cpanm again to pull in all dependencies for MogileFS. Read the
output carefully in case of any build failures.
```
cpanm IO::AIO MogileFS::Server MogileFS::Client MogileFS::Utils
```

Save this script as `mogstoreds.sh`. Edit anything necessary.
It is for starting and stopping a set of test mogstored's. For convenience
in testing replication, the HOSTS example below splits them into the
space of two /24's.
```
cat > mogstoreds.sh <<END
#!/bin/bash

BASEDIR="/home/mogilefs/mogdata/"
PERL5LIB="$PERL5LIB"
MOGSTORED="mogstored"

HOSTS="127.0.0.20 127.0.0.25 127.0.15.5 127.0.15.10"

if [ $1 = "start" ] ; then
    echo "Starting a few mogstored's"
    for host in $HOSTS ; do
        echo "starting host $host"
        $MOGSTORED -d --httplisten $host:7500 --mgmtlisten $host:7501 \
            --docroot ${BASEDIR}${host}
    done
elif [ $1 = "stop" ] ; then
    echo "Stopping mogstored's"
    for host in $HOSTS ; do
        echo "shutdown" | nc $host 7501
    done
fi
END
$ chmod +x mogstoreds.sh
```

Pre-make some directories and data
```
mkdir -p \
  mogdata/{127.0.0.20/{dev1,dev2},127.0.0.25/{dev3,dev4},127.0.15.5/{dev5,dev6},127.0.15.10/{dev7,dev8}}
```

Install sysstat if you'd like (provides iowait stats)
```
apt-get install sysstat
```

Prepare a database, and install it via `mogdbsetup`. Tune the below if you
know better. Run `mogdbsetup --help` for full options.
```
mysql> CREATE DATABASE mogilefs; GRANT ALL PRIVILEGES ON mogilefs.* TO \
'mogile'@'%' IDENTIFIED BY 'mogilepass';
# I don't know postgres very well ;)
mogdbsetup --dbhost=127.0.0.1 --dbname=mogilefs --dbuser=mogile \
--dbpass=mogilepass
# Follow any dialogs.
```

Prepare a tracker configuration.
```
cat > mogtracker.conf <<END
db_dsn = DBI:mysql:mogilefs:host=127.0.0.1
db_user = mogile
db_pass = mogilepass
listen = 127.0.1.5:7001
query_jobs = 2
delete_jobs = 1
replicate_jobs = 1
reaper_jobs = 1
fsck_jobs = 1
rebalance_ignore_missing = 1
END
```

Fire up some mogstored's.
```
./mogstoreds.sh start
```

Fire up a mogilefs tracker. Recommend for testing that you run it in the
foreground, or in a screen session, so you can quickly and easily see errors.
```
mogilefsd --config=./mogtracker.conf
```

Create a client utility configuration file.
```
echo "trackers = 127.0.1.5:7001
domain = toast" >> ~/.mogilefs.conf
```

Add the mogtracker hosts and devices
```
mogadm host add nearone --ip=127.0.0.20 --status=alive
mogadm host add neartwo --ip=127.0.0.25 --status=alive
mogadm host add farone --ip=127.0.15.5 --status=alive
mogadm host add fartwo --ip=127.0.15.10 --status=alive
mogadm device add nearone 1
mogadm device add nearone 2
mogadm device add neartwo 3
mogadm device add neartwo 4
mogadm device add farone 5
mogadm device add farone 6
mogadm device add fartwo 7
mogadm device add fartwo 8
```

Create a domain to test with
```
mogadm domain add toast
```

Wait a few seconds, then see if everything checks out
```
mogadm check
```

Try `mogupload`'ing something
```
echo "Hello, world" | mogupload --key="/hellothere" --file="-"
```

`mogfetch` it back
```
mogfetch --key="/hellothere" --file="-"
```

Yay, congrats!

