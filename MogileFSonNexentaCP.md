# Introduction #
This guide shows you how to set up a dedicated MogileFS storage server on Nexenta CP. As of MogileFS 2.36 MogileFS runs great on Nexenta CP 2 and 3. I've tested this on Nexenta CP 2 and it probably works on Opensolaris as well. This will only provide a storage server. You still need a tracker box somewhere. This is unsupported by the core MogileFS devs and you should probably understand how it works before you deploy it.

The main reason for running MogileFS on Solaris is to use ZFS for storing files. ZFS checksums all the data so bit-rot can be found.

In the following section, we will setup nginx and mogstored Nexenta CP. We use nginx for get/put and use mogstored for usage, iostat, and FSCK fid stat'ing.

## Dependencies 

You'll need libwww-perl, libdbd-mysql-perl, zip, unzip, libprce-dev, and zlib-dev.
```
apt-get install libwww-perl libdbd-mysql-perl zip unzip libprce-dev zlib-dev
```

You also need [Nginx](http://nginx.org/en/download.html) I recommend the 1.0 stable branch



## Installation 

1) Setup your zpools. You can do a single zpool per disk (no redundancy but zfs will notify you when a file is corrupt), or you can setup a raid type zpool. Either way the zpool must be named devXXX for the mogile device id you're using and must be mounted at /var/mogdata/devXXX. You can't have any children filesystems in the zpool.
Sample zpool creation and zfs creation
```
zpool create dev58 c0t2d0
zfs set mountpoint=/var/mogdata/dev58 dev58
```

This results in a zfs filesystem that displays like this

```
# zfs list dev58
NAME    USED  AVAIL  REFER  MOUNTPOINT
dev58   317G  1.47T   317G  /var/mogdata/dev58
```

2) Add the nginx user and group. Everything will run as the nginx user. Don't change the home directory unless you want to fix the smf files by hand.

```
groupadd nginx
useradd -g nginx -d /opt/nginx nginx
```

3) Install nginx. I used the latest stable from the 1.0 series. I compiled it with the following command. It puts everything into /opt/nginx.
```
./configure  --user=nginx --group=nginx --prefix=/opt/nginx --sbin-path=/opt/nginx/sbin/nginx 
--conf-path=/opt/nginx/etc/nginx.conf --error-log-path=/opt/nginx/log/error.log 
--http-log-path=/opt/nginx/log/access.log --http-client-body-temp-path=/opt/nginx/tmp/client_body 
--http-proxy-temp-path=/opt/nginx/tmp/proxy --http-fastcgi-temp-path=/opt/nginx/tmp/fastcgi 
--pid-path=/opt/nginx/nginx.pid --lock-path=/opt/nginx/nginx-lock --with-http_dav_module 
--with-http_stub_status_module

make; make install
```


4) Create a couple directories and set permissions on them
```
mkdir -p /opt/nginx/tmp
mkdir -p /var/mogdata
chown -R nginx:nginx /var/mogdata
chown -R nginx:nginx /opt/nginx
```


5) Install MogileFS from cpan. (ie 'cpan> install MogileFS::Utils' and 'cpan> install MogileFS::Server')

6) Setup an /etc/mogilefs/mogstored.conf file that looks like this. We won't use the httplisten port but you could if you wanted to.

```
httplisten=0.0.0.0:7600
mgmtlisten=0.0.0.0:7501
docroot=/var/mogdata
server=perlbal
```

7) Setup your nginx.conf file in /opt/nginx/etc/nginx.conf. Here's one I use

```
user              nginx;
worker_processes  2;
error_log         /opt/nginx/log/error.log crit;
pid               /opt/nginx/nginx.pid;

events {
    worker_connections  1024;
}

http {
        include /opt/nginx/etc/mime.types;
        access_log /dev/null;
        #access_log /opt/nginx/log/access_log;
        default_type application/octet-stream;
        sendfile on;
        keepalive_timeout 30;
        tcp_nodelay on;
        client_max_body_size 251m;
        server_tokens off;
        server {
                listen 7500;
                server_name NAME_OR_IP_HERE;

                charset utf-8;
                location /devXXX {
                        root /var/mogdata/;
                        dav_methods put delete copy move;
                        dav_access user:rw group:rw all:r;
                        create_full_put_path on;
                        client_body_temp_path /var/mogdata/devXXX/.nginxtmp 1 2;
                }
                location /devXXX {
                        root /var/mogdata/;
                        dav_methods put delete copy move;
                        dav_access user:rw group:rw all:r;
                        create_full_put_path on;
                        client_body_temp_path /var/mogdata/devXXX/.nginxtmp 1 2;
                }
                location /devXXX {
                        root /var/mogdata/;
                        dav_methods put delete copy move;
                        dav_access user:rw group:rw all:r;
                        create_full_put_path on;
                        client_body_temp_path /var/mogdata/devXXX/.nginxtmp 1 2;
                }
        }
}
```

Alternatively here is a script I use to automatically generate an nginx conf file. It will find all your zpools called devXXX and build the conf file. Run it like this mognginxsetup > /opt/nginx/etc/nginx.conf

```
#!/usr/bin/perl

## Written by Daniel Leaberry <leaberry@gmail.com>
## Code is GPL3

## ip based server hostnames work best on a DNS free network
## You may need to adjust the grep command to find your ip address
$ipaddr=`ifconfig -a \| grep inet \| grep ffff \| awk \'\{print \$2\}\'`;


## Creates an nginx.conf config that includes each devXX and the
## client_body_temp_path is set for each individual drive. Needs
## to be re-ran after any devices are changed (ie dead and replaced)

my $nginx_tail = <<"EOF";
	}
}
EOF

 
my $nginx_head = <<"EOF";
######################################################################
# More information about the configuration options is available on
#   * the English wiki - http://wiki.codemongers.com/Main
#######################################################################

#----------------------------------------------------------------------
# Main Module - directives that cover basic functionality
#
#   http://wiki.codemongers.com/NginxMainModule
#
#----------------------------------------------------------------------

user              nginx;
worker_processes  2;
error_log         /opt/nginx/log/error.log crit;
pid               /opt/nginx/nginx.pid;

#----------------------------------------------------------------------
# Events Module
#
#   http://wiki.codemongers.com/NginxEventsModule
#
#----------------------------------------------------------------------
events {
    worker_connections  1024;
}
#----------------------------------------------------------------------
# HTTP Core Module
#
#   http://wiki.codemongers.com/NginxHttpCoreModule
#
#----------------------------------------------------------------------
http {
        include /opt/nginx/etc/mime.types;
        access_log /dev/null;
	#access_log /opt/nginx/log/access_log;
	default_type application/octet-stream;
        sendfile on;
        keepalive_timeout 0;
        tcp_nodelay on;
        client_max_body_size 251m;
        server_tokens off;
        server {
                listen 7500;
                server_name $ipaddr;
                charset utf-8;
EOF



##############################################################
##############################################################
##############################################################


$hostname=`hostname -s`;
chomp($hostname);

open(MOUNTS, "zfs list -Ho mountpoint |");
@mounts=<MOUNTS>;
close(MOUNTS);

print $nginx_head;

foreach $line (@mounts)
{
	if( $line=~/.*dev(\d+)/ )
	{
		print <<"EOF";
		location /dev$1 {
			root /var/mogdata/;
			dav_methods put delete copy move;
			dav_access user:rw group:rw all:r;
			create_full_put_path on;
			client_body_temp_path /var/mogdata/dev$1/.nginxtmp 1 2;
		}
EOF
	}
}
print $nginx_tail;

```



8) Import the smf scripts to make everything run on startup. All servers run as the nginx user.

```
svccfg import nginx.smf
svccfg import mogstored.smf
```


---

nginx.smf
```
<?xml version="1.0"?>
<!DOCTYPE service_bundle SYSTEM "/usr/share/lib/xml/dtd/service_bundle.dtd.1">

<!--
-->

<service_bundle type="manifest" name="nginx">
 <service name="mogilefs/nginx" type="service" version="0">
   <create_default_instance enabled="true"/>
   <single_instance/>

   <dependency name="net" grouping="require_all" restart_on="none" type="service">
        <service_fmri value="svc:/milestone/network:default"/>
        <service_fmri value="svc:/system/filesystem/local:default"/>
        <service_fmri value="svc:/network/shares/group:zfs"/>
   </dependency>

   <exec_method name="start" type="method" exec="/opt/nginx/sbin/nginx -c /opt/nginx/etc/nginx.conf" timeout_seconds="60">
     <method_context working_directory="/opt/nginx">
      <method_credential user="nginx" group="nginx" privileges="basic,net_privaddr"/>
      <!--method_credential user="root" group="root" /-->
       <method_environment>
         <envvar name="PATH" value="/opt/nginx/sbin:/usr/bin:/bin:/opt/nginx" />
       </method_environment>
     </method_context>
   </exec_method>

   <exec_method name="stop" type="method" exec="/usr/bin/pkill -QUIT nginx" timeout_seconds="60">
     <method_context working_directory="/opt/nginx">
      <method_credential user="nginx" group="nginx" privileges="basic,net_privaddr"/>
      <!--method_credential user="root" group="root" /-->
       <method_environment>
         <envvar name="PATH" value="/opt/nginx/sbin:/usr/bin:/bin:/opt/nginx" />
       </method_environment>
     </method_context>
   </exec_method>

   <exec_method name="restart" type="method" exec="/opt/nginx/sbin/nginx -c /opt/nginx/etc/nginx.conf" timeout_seconds="60">
     <method_context working_directory="/opt/nginx">
      <method_credential user="nginx" group="nginx" privileges="basic,net_privaddr"/>
      <!--method_credential user="root" group="root" /-->
       <method_environment>
         <envvar name="PATH" value="/opt/nginx/sbin:/usr/bin:/bin:/opt/nginx" />
       </method_environment>
     </method_context>
   </exec_method>

 </service>
</service_bundle>
```

---

mogstored.smf
```
<?xml version="1.0"?>
<!DOCTYPE service_bundle SYSTEM "/usr/share/lib/xml/dtd/service_bundle.dtd.1">

<!--
-->

<service_bundle type="manifest" name="mogstored">
 <service name="mogilefs/mogstored" type="service" version="0">
   <create_default_instance enabled="true"/>
   <single_instance/>

   <dependency name="net" grouping="require_all" restart_on="none" type="service">
        <service_fmri value="svc:/milestone/network:default"/>
        <service_fmri value="svc:/system/filesystem/local:default"/>
        <service_fmri value="svc:/network/shares/group:zfs"/>
   </dependency>

   <exec_method name="start" type="method" exec="/usr/local/bin/mogstored -d" timeout_seconds="60">
     <method_context working_directory="/opt/nginx">
      <method_credential user="nginx" group="nginx" privileges="basic,net_privaddr"/>
      <!--method_credential user="root" group="root" /-->
       <method_environment>
         <envvar name="PATH" value="/opt/nginx/sbin:/usr/bin:/bin:/sbin:/usr/sbin:/opt/nginx:/usr/local/bin" />
       </method_environment>
     </method_context>
   </exec_method>

   <exec_method name="stop" type="method" exec="/usr/bin/pkill -QUIT mogstored" timeout_seconds="60">
     <method_context working_directory="/opt/nginx">
      <method_credential user="nginx" group="nginx" privileges="basic,net_privaddr"/>
      <!--method_credential user="root" group="root" /-->
       <method_environment>
         <envvar name="PATH" value="/opt/nginx/sbin:/usr/bin:/bin:/sbin:/usr/sbin:/opt/nginx" />
       </method_environment>
     </method_context>
   </exec_method>

   <exec_method name="restart" type="method" exec="/usr/local/bin/mogstored -d" timeout_seconds="60">
     <method_context working_directory="/opt/nginx">
      <method_credential user="nginx" group="nginx" privileges="basic,net_privaddr"/>
      <!--method_credential user="root" group="root" /-->
       <method_environment>
         <envvar name="PATH" value="/opt/nginx/sbin:/usr/bin:/bin:/sbin:/usr/sbin:/opt/nginx:/usr/local/bin" />
       </method_environment>
     </method_context>
   </exec_method>

 </service>
</service_bundle>
```

---


9) You should see usage files in /var/mogdata/devXXX/usage and if you telnet to localhost 7501 and issue "watch" it will start spitting out the statistics the tracker needs.

10) Add the host and devices to mogile like you normally would. From the trackers point of view the storage node is a completely normal mogile storage node.

11) You can also run a tracker if you want. I haven't extensively tested it but it does work.
