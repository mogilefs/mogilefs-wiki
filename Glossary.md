# Glossary

## domain 

A domain is the top level separation of files. File keys are unique within domains. A domain consists of a set of classes that define the files within the domain. Examples of domains: fotobilder, livejournal.

## class 

Every file is part of exactly one class. A class is part of exactly one domain. A class, in effect, specifies the minimum replica count of a file. Examples of classes: userpicture, userbackup, phonepost. Classes may have extra replication policies defined.

## key 

A key is a unique textual string that identifies a file. Keys are unique within domains. Examples of keys: userpicture:34:39, phonepost:93:3834, userbackup:15. Fake structures work too: /pics/hello.png, any string.

## minimum replica count (mindevcount) 

This is a property of a class. This defines how many times the files in that class need to be replicated onto different devices in order to ensure redundancy among the data and prevent loss.

## file 

A file is a defined collection of bits uploaded to MogileFS to store. Files are replicated according to their minimum replica count. Each file has a key, is a part of one class, and is located in one domain. Files are the things that MogileFS stores for you.

## fid 

A fid is an internal numerical representation of a file. Every file is assigned a unique fid. If a file is overwritten, it is given a new fid.