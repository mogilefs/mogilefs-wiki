# Simple Application Example #

If you fetch the [MogileFS::Server](http://search.cpan.org/dist/MogileFS-Server/) tarball from 2.46 or later, you will find an
example/testapp/ directory with a few files in it. With this you will see how
a barebones application would serve up files from MogileFS.


## Overview Reminder ##

Before we begin, a quick reminder of the flow for how applications should
serve files from MogileFS:

  * Browser issues "GET /foo" request to a perlbal load balancer
  * Perlbal routes the request to your application server (perl, php, ruby, etc)
  * Application asks a MogileFS tracker where the file is (or cache/etc).
  * Application returns magic 'X-REPROXY-URL: http://path/from/mogilefs/tracker' header.
  * Perlbal intercepts that header, and responds to the original Browser with the actual file.

This quick reminder to why it's done this way is for scalability, efficiency,
and flexibility. The application server doesn't have to read the file and then
return it. It has to find the paths and can then service other requests.

Perlbal can take its sweet time in returning data to the client. Perlbal could
also remember the request and respond with the file without ever talking to
the application. Also, for many other reasons we won't get into now :)

If you don't understand any of this, just read on below and see how it's done
in practice.

## Requirements ##

Requirements for the example application are a working MogileFS cluster, as
you'd get from following [InstallHowTo](InstallHowTo.md). You should also be familiar with
[CommandlineUsage](CommandlineUsage.md), so you can upload and test files in MogileFS easily.

Finally, you need Plack installed for actually running the application.
Install it the same way you've pulled in MogileFS::Server and friends.

`cpanm Plack` is a great an easy way to do this.

You might also need to install Perlbal::XS::HTTPHeaders to get perlbal to
start. If you cannot do this, simply comment out the `XS enable headers` line
from testapp-perlbal.conf


---

# Starting Up Perlbal #

First, review the testapp-perlbal.conf file. It has a few things that you may
ignore, but a few things are key:

```
CREATE POOL testapp_pool
  POOL testapp_pool ADD 127.0.0.1:5000
```

This is where your application server will live. If this were a big website
you'd have many POOL lines, one for each webserver.

```
  SET role = reverse_proxy
  SET pool = testapp_pool 
  SET listen = 0.0.0.0:7070
  SET enable_reproxy = on
```

These lines are the meat, defining that perlbal will listen on port 7070, and
that reproxies (what mogilefs relies on) should be allowed.

To start up perlbal with this configuration:

```
$ perlbal -c examples/testapp/testapp-perlbal.conf
```

(note that for simplicity of testing you should leave these running in the
foreground for now. use a few terminal sessions if you have to).


---

# Starting The Application #

The test app is extremely simple. It only talks in plain text and will only
serve files as plain text.

First, open up testapp.psgi and edit two lines near the top.

```
my $TRACKERS = ['tracker1:7001'];
my $DOMAIN   = 'toast';
```

Replace these with what your tracker hostname/port actually is, and what your
test domain actually is.

Assuming you have Plack installed already, and nothing else running on port
5000, starting it should be simple:

```
$ plackup examples/testapp/testapp.psgi
HTTP::Server::PSGI: Accepting connections at http://0:5000/
```


---

# Uploading a File #

[CommandlineUsage](CommandlineUsage.md) goes into great depth on this. For simplicity:

```
$ echo "hello I am a test file" | mogupload --key="/hellotest" --file="-"
```


---

# Fetching the File #

If you were to attempt to navigate to http://127.0.0.1:5000/hellotest - your
browser would get a cryptic empty response. If your application is designed
better than this test app, it shouldn't even be possible to get a response
unless first going through perlbal.

Now, fire up `http://127.0.0.1:7070/hellotest' in your browser. Replace
`127.0.0.1` as necessary if the test host is not local to you.

If all went well you should be greeted with "Hello I am a test file"!


---

# Understanding the App #

Now that you have it all running, you should take some time to read through
the source of testapp.psgi, and try to understand the Perlbal configuration a
bit. After Perlbal 1.77, it has a full manual included (`perlbal
Perlbal::Manual` to see it).

The testapp.psgi file is heavily commented to explain each step. It is
stripped down to be very barebones. For a real application you should use a
full framework, and especially try to avoid relying on raw Plack :)

If you run into any trouble or have any questions, feel free to ask for help
on the mailing list or #mogilefs on freenode IRC.