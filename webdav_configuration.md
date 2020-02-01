# Using Nginx/Apache2 as a Storage Server
Alternatively, you can use other webdav servers to serve files instead of build-in one in mogstored.

## Apache2

```
<VirtualHost 0.0.0.0:7500>
    DocumentRoot /home/mog-data
    <Location />
        Dav On
        ContentDigest on
    </Location>
    <Directory /home/mog-data>
        Options +Indexes +FollowSymLinks
	Require all granted
    </Directory>
    ContentDigest on
    ErrorLog /var/log/apache2/error.log
    # Possible values include: debug, info, notice, warn, error, crit, alert, emerg.
    LogLevel debug
    # CustomLog /var/log/apache2/access.log combined
</VirtualHost>
```

## Nginx

### nginx configuration file - separate port for GET requests ##

```
server
{
#       port to listen on for GET requests. You can make it bind() to specific ip with
#       listen 123.123.123.123:80, also you can specify several "listen" lines.
        listen                                  80;
#       if you don't want access log - comment this line
        access_log                              /var/log/nginx.access.log;
#       where to log errors
        error_log                               /var/log/nginx.error.log warn;

        location /
        {
#               root path of your MogileFS files
                root                            /var/mogdata;
#               adds Expires HTTP header. Usefull if you serve static content that don't
#               change over time and have permanent links (image sharing service for
#               example)
                expires                         max;
        }
}
server
{
        listen                                  7500;
        error_log                               /var/log/nginx.error.log crit;

        location /
        {
#               this line is very important - you will get errors if you miss it
                autoindex                       on;
                root                            /var/mogdata;
        }
#       a location directive per device. you need this to make nginx store temporary
#       file on the same device where the upload will be stored
        location /dev101/
        {
                root                            /var/mogdata;
#               maximum upload size
                client_max_body_size            20m;
#               this needs to be on the same device with '/var/mogdata/dev101' so nginx
#               can move uploaded file instantly
                client_body_temp_path           /var/mogdata/dev101/temp;
#               MogileFS uses only this WebDAV methods
                dav_methods                     PUT DELETE;
                create_full_put_path            on;
                dav_access                      user:rw  group:r  all:r;
#               Non-GET requests are allowed only for 192.168.0.0/16
                limit_except GET {
                    allow                       192.168.0.0/16;
                    deny                        all;
                }
        }
        location /dev102/
        {
                root                            /var/mogdata;
                client_max_body_size            20m;
                client_body_temp_path           /var/mogdata/dev102/temp;
                dav_methods                     PUT DELETE;
                create_full_put_path            on;
                dav_access                      user:rw  group:r  all:r;
                limit_except GET {
                    allow                       192.168.0.0/16;
                    deny                        all;
                }
        }
}
```

### nginx configuration file - single port for GET and WebDAV ##

```
server
{
        listen                                  7500;
        error_log                               /var/log/nginx.error.log crit;

        location /
        {
                autoindex                       on;
                root                            /var/mogdata;
        }
        location /dev101/
        {
                expires                         max;
                root                            /var/mogdata;
                client_max_body_size            20m;
                client_body_temp_path           /var/mogdata/dev101/temp;
                dav_methods                     PUT DELETE;
                create_full_put_path            on;
                dav_access                      user:rw  group:r  all:r;
                limit_except GET {
                    allow                       192.168.0.0/16;
                    deny                        all;
                }
        }
        location /dev102/
        {
                expires                         max;
                root                            /var/mogdata;
                client_max_body_size            20m;
                client_body_temp_path           /var/mogdata/dev102/temp;
                dav_methods                     PUT DELETE;
                create_full_put_path            on;
                dav_access                      user:rw  group:r  all:r;
                limit_except GET {
                    allow                       192.168.0.0/16;
                    deny                        all;
                }
        }
}
```
