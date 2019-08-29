Cavium ThunderX Operations Manual
=================================

Cluster Overview
----------------

### Hardware

20 compute nodes (Gigabyte [model number]):
* 2x 48 core Cavium ThunderX processors (96 cores)
* 500 GiB RAM
* 12x 8 TB Hard drives
* 40 GbE network connection

3 head nodes: [TODO: fill in details]

1 login node: [TODO: fill in details]

Further, each compute node has its first hard drive (at `/dev/sda`) partitioned
in the following manner: [TODO: fill in details]

Head nodes partition info: [TODO: fill in details]

Each of the remaining drives in the compute nodes have one xfs formatted
partition completely dedicated to HDFS.  Additionally, each compute node is
configured to yield 90 cores and 450 GiB of memory to YARN for computation.
This gives the cluster the following specifiations:

*Cores*: 1800
*Memory*: 8.79 TiB
*HDFS*: ~1.7 PB

### Software

* Distributed storage: HDFS
* Resource manager: YARN
* Client applications
    * MapReduce
    * Spark (PySpark, SparkR)
    * Hive (Hive-on-MapReduce)

The software stack is build using [Apache Bigtop](http://bigtop.apache.org/).
We have our HDFS configured to use HA with two NameNodes (active and standby).
As such, all three head nodes run the following:

* ZooKeeper server daemon
* Hadoop JournalNode daemon

Of the three head nodes, two are designated as Hadoop NameNodes, and each of
those runs the following:

* Hadoop NameNode daemon
* ZooKeeper failover controller daemon (ZKFC)

The third head node is designated as our resource manager server and runs the
following:

* YARN ResourceManager daemon

It is also designated to be our "true" head node.  This entails two things:

* It is considered the source of truth for user management purposes (i.e. the
  `/etc/passwd` file is the master copy)
* It can write to all Cavium-specific mounts as `root`

All of the compute nodes run the following:

* Hadoop DataNode daemon
* YARN NodeManager daemon

All software installation and configuration is done via
[Ansible](https://www.ansible.com) and is version controlled.

Daily Tasks
-----------

Common Tasks
------------

Common tasks often include configuration changes via Ansible.  All steps will
assume that you have cloned the ARC-TS Ansible repo to a local space.  If you
have not done this, you can do so via the following command on your local
machine:  
``` 
$ git clone git@bitbucket.org:umarcts/ansible.git
```

All paths assume that your current working directory is `ansible` (i.e. your
locally cloned Ansible repo).

Lastly, it is suggested that all pushses of configuration changes be as yourself
using sudo from `/etc/ansible` on flux-admin09.

### Add users

To add a new user to the cluster, `ssh` to cavium-rm01 (the designated head
node) and run the following script:  
```
$ /usr/arcts/systems/scripts/create-cavium-hadoop-user.sh -u $uniqname -g $grp
```

For the `$grp`, I (pattonmj) usually try to add them to the group based on their
school.  If they have a Flux account, you can get the primary group from there.
If all else fails, add them to the `workshop` group.

Once this is done, add the user to the [group] MCommunity group so that they
can get emails. 

### Add mounts

If a user requests to have a MiStorage, Turbo, or Locker mount added to the
cluster:

1. Create a hotfix branch named after the request.  If the request came in
   ticket, include the ticket number in the branch name.  
```
$ git checkout -b hotfix/cavium-autofs-fp12345
```

2. Edit `group_vars/caviumHadoop/autofs`.  Use the current entries to determine
   how the file is formatted and which mount options to use.
    a. If the mount is MiStorage, add it to the `auto.nfs` content section.
    b. If the mount is Turbo, add it to the `auto.nfs.turbo` content section.
    c. If the mount is Locker, add it to the `auto.nfs.locker` content section.

3. Commit the changes.

4. Push the hotfix branch to origin.  
```
$ git push origin hotfix/cavium-autofs-fp12345
```

5. Create a pull request (PR) on bitbucket.org.

6. Once the PR has been approved, merged, and pulled to flux-admin09, push the
   change to the cluster.
```
$ sudo ansible-playbook -i dsi-prod/ -l caviumHadoop -t 'autofsMountUpdate' utilityPlaybook/autofs.yml
```

### Add Python libraries (deprecated)

If a user requests some Python library to be installed:

1. Edit `includes/install_python_libs.yml`.  Be sure to add the version of the
   package that you are installing (this can be found easily enough using `pip
   search $package`).  Add the package under the appropriate stanzas (for Python 
   2 and 3 if both packages are available).

2. See steps 3-5 of 'Add mounts' section.

3. Once the PR has been approved, merged, and pulled to flux-admin09, push the
   change to the cluster.  
```
$ sudo ansible-playbook -i dsi-prod/ -l caviumHadoop -t 'install_python_libs' hadoop-playbook.yml
```

### Add new queues

YARN queues are defined in (and distributed via) Ansible.  Currently, there are
queues for each major research group that we support.  You can find the queue
definitions in `dsi-prod/group_vars/caviumHadoop/queues`  There is actually only
a single dictionary defined there calles `queues`.  The keys of the dictionary
are the names of the queues, and the values are the queues' configurations.
Using the `default` queue as an example, we have:

```
queues:
  default:
    capacity: 5
    max_capacity: 5
    state: "RUNNING"
    users:
      - "*"
    admin_users:
      - pattonmj
      - smeyer
```

The following are the only values that are _required_:

* `capacity`: expressed as a percentage of the cluster (float)
* `state`: should be either `"RUNNING"` or `"STOPPED"`, including the quotes
  (string)
* `users`: list of uniqnames in YAML list syntax, even if there is only one user
  (list)
* `admin_users`: list of uniqnames in YAML list syntax, even if there is only
  one user (list)

Other configuration values currently supported are:

* `max_capacity`: for queue elasticity; allows a queue to use more than its
  configured capacity.  Expressed as a percentage of the cluster (float).
* `user_limit_factor`: how much of the queue a single user can consume,
  expressed as a multiplier (float)
* `subqueues`: list of queues in YAML list syntax that split the current queue's
  resources, even if there is only one (list)

Note that subqueues need to be defined in the same way that regular queues are.
The only difference is that their names should be formatted as such:
`$parent_queue.$subqueue`.  For example, if we wanted to add a subqueue called
`special` to the `default` queue, we would define it under `queues` in this way:  

```
queues:
.
.
.
  default.special:
    capacity: 20
    state: "RUNNING"
    users:
      - pattonmj
    admin_users:
      - pattonmj
      - smeyer
```

Finally, it is important to note that all of the `capacity` values for queues
*_at the same level_* should add to 100 (i.e. 100%).  The main consequence of
this is that, when adding a new queue, the capacities of all other queues *_at
the same level_* need to be recalculated.  If this is not done, refreshing the
queue configuration will fail.

Please see the Apache documentation for the [YARN Capacity
Scheduler](https://hadoop.apache.org/docs/r2.8.4/hadoop-yarn/hadoop-yarn-site/CapacityScheduler.html)
if more detail is necessary.

The full steps to add a new queue follow:

1. Edit `dsi-prod/group_vars/caviumHadoop/queues` using the configuration
   guidelines described above.

2. See steps 3-5 of the 'Add mounts' section.

3. Once the PR has been approved, merged, and pulled to flux-admin09, push the
   change to the cluster.  
```
$ sudo ansible-playbook -i dsi-prod/ -l 'caviumHadoop' -t 'update-queues' hadoopUtilityPlaybooks/yarn-queues.yml
```

### Add users to queues

Everyone with a Cavium ThunderX account has access to the default queue.  If a
user needs to be added to a certain YARN queue:

1. Edit `dsi-prod/group_vars/caviumHadoop/queues`.  Add the user's uniqname to
   the appropriate queue under the queue's `users` key.

2. See steps 3-5 of the 'Add mounts' section.

3. Once the PR has been approved, merged, and pulled to flux-admin09, push the
   change to the cluster.  
```
$ sudo ansible-playbook -i dsi-prod/ -l caviumHadoop -t 'update-queues' hadoopUtilityPlaybooks/yarn-queues.yml
```

Maintenance
-----------

During maintenance, one of the most common tasks we perform is to update the OS.
This can readily be done using Ansible.  There are a number of steps involved,
however, some of which should be done prior to the maintenance window.  

### Use `reposync` to get OS updates (pre-maintenace)

The first task should be to pull down the newest packages.  Using CentOS, only
the `updates` repo needs to be pulled.  The following instructions assume you
are on flux-admin09 as root.

1. Create a directory for the version that you are pulling.  
```
$ mkdir /usr/arcts/repos/dist/CentOS/$arch/$version
```

For the Cavium ThunderX cluster, `$arch` will always be `aarch64`.  (It could be
`x86_64` if working on other systems.)

The ARC-TS convention for versioning is to use the OS major and minor versions,
but then the date of when you are pulling the repo.  Thus the format for a
version is `$major.$minor.$yymm`.  For example, if you are just updating CentOS
7.6 but you are doing it in August 2019, your `$version` would be `7.6.1908`.

2. Sync the `updates` repo.  
```
$ reposync -t -a aarch64 \
  --repoid=updates
  -c /usr/arcts/repos/reposyncFiles/CentOS-7_aarch64/local_yum.conf \
  -p /usr/arcts/repos/dist/CentOS/aarch64/$version/
```

It is also possible to use the `-n` flag of `reposync` to get only the newest
packages.

3. Create symlinks inside of the new `$version` directory to point to older
   repos.  
```
$ ln -s /usr/arcts/repos/dist/CentOS/aarch64/$prev_version/os/ \
  /usr/arcts/repos/dist/CentOS/aarch64/$version/os

$ ln -s /usr/arcts/repos/dist/CentOS/aarch64/$prev_version/extras/ \
  /usr/arcts/repos/dist/CentOS/aarch64/$version/extras
```

Here, `$prev_version` is the next newest version that is available.  For
example, if `$version` is `7.6.1908`, `$prev_version` _might_ be `7.6.1901` if
the last update was performed in January 2019.  It is necessary to create these
symlinks because Ansible will deploy repo files based on what you use as
`$version`.  If those links are not present (which they won't be if you only
sync the updates repo), `yum` transactions will fail because they can't find the
specified repos.

### Update the version variable in Ansible (pre-maintenance)

Winter 2019 Maintenance Changes
-------------------------------

* Update to CentOS 7.6
* Kerberization completed
* Increased available resources for compute nodes
    * Increased cores available to YARN from 80 to 90
    * Increased memory available to YARN from 400 GiB to 450 GiB

### Operational Changes

Due to a bug in the Bigtop 1.2 software stack, starting secure DataNodes using a
service manager (i.e. systemd) no longer works.  The service reports failure, so
child processes are killed.  The DataNode service must be started manually.  It
still _reports_ failure if done this way, but using `ps` reveals that the
process is actually running.  As such, there is a new procedure to shutdown and
start services on DataNode (compute) servers.

1. Stop the NodeManager service as normal  
```
$ systemctl stop hadoop-yarn-nodemanager.service
```

2. Stop the DataNode service by killing its processes (there should be two):
```
$ for pid in $(ps -ef | grep datanode | awk '{print $2}'); do kill $pid; done
```

To start services on compute servers:

1. Start the DataNode service with the following command as `root`  
```
$ . /etc/hadoop/conf/hadoop-env.sh && /etc/init.d/hadoop-hdfs-datanode start
```

2. Start the NodeManager service as normal  
```
$ systemctl start hadoop-hdfs-nodemanager.service
```

A major consequence of this is that DataNode logs are now in a different place.
They can be found in `/usr/lib/hadoop/logs/` instead of `/var/log/hadoop-hdfs/`.

Further, we must now obtain Kerberos tickets to run administrative `hdfs` and
`yarn` commands.  There are keytabs in place for this.  For hdfs (as the `hdfs`
user):  
```
$ kinit -kt /etc/security/keytabs/hdfs.service.keytab hdfs-cavium1
```

For YARN (as the `yarn` user):  
```
$ kinit -kt /etc/security/keytabs/rm.service.keytab rm/cavium-rm01.arc-ts.umich.edu
```

Also, if you wish to run jobs as yourself, you have two options:  

1. Log directly into the login server from your machine.  All things should work
   as normal.

2. `ssh` to the login server as `root` from flux-admin09, then:
    a. `su` to yourself
    b. run `kinit` (with no arguments)
    c. enter your UMICH password


