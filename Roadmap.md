# MogileFS Roadmap #

High level overview of future plans/ideas. There is no set schedule and
everything listed here is up for debate, refinement, addition, or could be
dropped.

If you have feedback or anything you'd like to see here, please start a thread on the mailing list.

The below are in no particular order and will likely lack implementation
details.

## Full featured perlbal plugin ##

It's pretty hard to get off the ground with MogileFS. First you set up the
server system, then your application needs to be tuned to look up and reproxy
files. A perlbal plugin should be provided with the distribution which is able
to directly serve files itself.

A preview of working code has been posted to the mailing list:
http://groups.google.com/group/mogile/browse_thread/thread/e3a33161d2da1d2a/00889fe6893fab21

## Documentation improvements ##

Some or all of the wiki should be moved into POD docs in the code, and then
the wiki generated off of that. Or at very least, a lot more of it needs to be
documented.

## Much more comprehensive tracker control/monitoring ##

A lot of the times we tell people to telnet to a tracker and run some
commands. Instead, `mogadm` should be able to centrally control jobs,
monitoring, etc. More of the tracker configuration should be moved out of
config files and into the database.

## File checksumming ##

This feature is implemented in 2.60 and refined in later releases.
See [Checksums](Checksums.md).

## JSON protocol overhaul ##

The protocol is flat and very manual. Replacing it with a JSON based protocol
would
allow a huge simplification protocol handling code. Many languages have JSON
libraries available by default now, so it would also make it simpler to port.

## Overhaul replication ##

Replication should be traceable, verifyable, and have a much better extension
API. Currently "ran out of ideas for replicating blah" is often the most
information you'll get.

## Out of system backups/replication ##

It's possible to add flexibility for how mogile handles deletes and updloads.
All changes could be replicated to a backup system, S3, etc.

## File versioning ##

It should be possible for all overwritten files to be versioned via a history
table. Making it innately possible to undo deletes or seek around in history,
useful for a backup system.

