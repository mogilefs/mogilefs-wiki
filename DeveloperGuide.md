# Developer Guide

Want to play with the MogileFS API? Want to help write code, test patches, fix
bugs? Head over to [Github](http://github.com/mogilefs/) (or if you're following a
link from the mailing list, go there instead). Pick a repository (such as
MogileFS-Server).

Make sure to stop any running instances you might've configured if you
wish to run `make test` safely.
```
./mogstoreds stop
echo "shutdown" | nc 127.0.0.1 7001 # kill the tracker. ctrl-c also works
```

If you're comfortable using git, check out the repository
```
git clone https://github.com/mogilefs/MogileFS-Server.git
```

Or you may simply click the "downloads" link in the upper right to get a fresh
tarball.

Untar it, and install or run tests via perl.
```
perl Makefile.PL
make
make test
make install
```

If you wish tests to run better, use a MySQL database
```
export MOGTEST_DBHOST="192.168.1.50"
export MOGTEST_DBPORT="3306"
export MOGTEST_DBUSER="mogtest"
export MOGTEST_DBPASS="mogtest"
export MOGTEST_DBNAME="mogtest"
export MOGTEST_DBTYPE="MySQL"
perl Makefile.PL
make
make test
```

You can easily run individual tests with `prove`
```
# Edit some file in t/* or lib/*
prove -l -v t/05-that-bug-you-fixed.t
```
