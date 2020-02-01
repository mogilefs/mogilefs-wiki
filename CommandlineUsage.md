

# How To Interact with MogileFS #

We'll start with introducing the commandline utilities as a method to
upload, fetch, and inspect files from MogileFS. 

---

## Tools of the Trade ##

The commandline utils are provided by the MogileFS::Utils package. If you have
`mogadm` you probably have these guys. Note that this wiki page requires
version 2.19 and higher.

All of the utilities have basic documentation in `pod` format. If you want to
learn about `mogupload`, you could run man or perldoc (depending on your
system)

```
$ man mogupload
... or...
$ perldoc mogupload
```

That gives you syntax, examples, etc.


---

## Upload and Download ##

Assuming you have a running MogileFS instance from following [InstallHowTo](InstallHowTo.md),
and have a domain created, you may start shoveling files to and from it.

For review, setting up a domain:

```
$ mogadm --trackers=host domain list
... review any domains you might already have...
```

```
$ mogadm --trackers=host domain add testdomain
$ mogadm --trackers=host class add testclass
... and we'll add a class to use later...
```

First, I recommend setting up a mogilefs.conf file somewhere, so you don't
have to specify the tracker and domain every time. Most of the utilities pull
from the same config file, so you can type a bit less.

```
$ echo -e "trackers = host\ndomain = testdomain" >> ~/.mogilefs.conf
$ cat ~/.mogilefs.conf
trackers = host
domain = testdomain
```

Now, pick a file to upload. Preferrably something smallish.

```
$ dd if=/dev/zero of=toastfile bs=1M count=1
```

We'll use the "mogupload" and "mogfetch" utilities to have some fun.

```
$ mogupload --key="/helloworld" --file="./toastfile"
... if all goes well, no output...
```

While we've said "/helloworld" here, a key may be anything. It could be
"helloworld" or "UUID:13583589233523589235" or "MD5:ABCDEFetc". Whatever makes
the most sense for your application.

You can also use STDIN.

```
$ mogupload --key="/hellostdin/" --file="-" < ./toastfile
```

Then get the file back:

```
$ mogfetch --key="/helloworld" --file="./toastfilefetched"
.. you probably have a ./toastfilefetched file now.
```

Also with STDOUT.

```
$ mogfetch --key="/helloworld" --file="-" > ./stdouttoast
```

Now lets go bananas and make a copy of the file with that new class!

```
$ mogfetch --key="/helloworld" --file="-" | mogupload --key="/hellocopy" \
    --class="testclass" --file="-"
```

... though if you just want to change the class of a file, there're ways of
doing that from client code.


---

## Deleting Files ##

Easy.

```
$ mogdelete --key="/goodbyeworld"
```


---

## Further Reading ##

There are other utilities for listing fids, finding metadata information, and
troubleshooting files. See `perldoc MogileFS::Utils` for an up-to-date list,
and `perldoc mogprogramname` for examples and details of the utility.


---

## Writing an Application ##

While using MogileFS as a personal filestore is fun, most of you are writing
applications. Since MogileFS is a non-POSIX distributed service, serving files
from it takes a little understanding.

Click on to [AppExample](AppExample.md) to see one in action.