**Warning: The repo seems out-of-date. Consider alternative ways please.**

Install on Ubuntu 
==============================

```
sudo apt-get install python-software-properties
sudo add-apt-repository ppa:gaod/mogilefs
```


  * Update the apt cache and install the mogile packages
```
sudo apt-get update
sudo apt-get install mogilefsd mogstored mogilefs-utils perlbal
```
  * Edit the configuration file(s) at `/etc/mogilefs/`
```
sudo dpkg-reconfigure mogilefsd
sudo dpkg-reconfigure mogstored
```
  * Set up your database
    * Database Host:
```
mysql> CREATE DATABASE mogilefs; 
mysql> CREATE USER 'mogilefs'@'mogiledb.yourdomain.loc' IDENTIFIED BY 'sekrit'; 
mysql> GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,ALTER ON mogilefs.* TO 'mogilefs'@'mogiledb.yourdomain.loc'; 
mysql>flush privileges;
```
    * Mogile Host use "mogdbsetup" (delivered with .deb):
```
mogdbsetup --dbhost=mogiledb.yourdomain.loc --dbname=mogilefs --dbuser=mogilefs --dbpassword=sekrit
```
  * Start the daemons
```
sudo /etc/init.d/mogilefsd start
sudo /etc/init.d/mogstored start
```
