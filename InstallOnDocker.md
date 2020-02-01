Using Docker to deploy MogileFS for test 
==============================
In this instruction, we will setup a mogilefs cluster for test.

We use [MAIO - MogileFS All In One](https://github.com/hrchu/mogilefs-all-in-one-docker) here since it provides a way to set up a minimal [MogileFS](https://github.com/mogilefs/mogilefs-wiki) cluster without pain. Tracker, stored and tracker DB are instantiated in a single container. It is suitable for testing and developing scenarios. 
 
## Howto

```
# docker pull hrchu/mogilefs-all-in-one
# docker run -e DOMAIN_NAME=testdomain -e CLASS_NAMES="testclass1 testclass2" -t -d -p 7001:7001 -p 7500:7500 --name maio hrchu/mogilefs-all-in-one
```
Done! MogileFS is ready to war on port 7001/7500! 😎

Note that in this deployment, we have the following configurations:
* domain: testdomain
* class: testclass1, testclass2

You can check it's status and connectivity by:
```
# echo '!jobs' |nc localhost 7001 
delete count 1
delete desired 1
delete pids 402
fsck count 1
fsck desired 1
fsck pids 405
job_master count 1
job_master desired 1
job_master pids 404
monitor count 1
monitor desired 1
monitor pids 399
queryworker count 5
queryworker desired 5
queryworker pids 406 407 408 409 410
reaper count 1
reaper desired 1
reaper pids 401
replicate count 1
replicate desired 1
replicate pids 403
.
```
or
```
# docker exec -it maio mogadm check
Checking trackers...
  127.0.0.1:7001 ... OK

Checking hosts...
  [ 1] mogilestorage ... OK

Checking devices...
  host device         size(G)    used(G)    free(G)   use%   ob state   I/O%
  ---- ------------ ---------- ---------- ---------- ------ ---------- -----
  [ 1] dev1           478.225     60.039    418.186  12.55%  writeable   N/A
  [ 1] dev2           478.225     60.039    418.186  12.55%  writeable   N/A
  ---- ------------ ---------- ---------- ---------- ------
             total:   956.450    120.078    836.372  12.55%
```

