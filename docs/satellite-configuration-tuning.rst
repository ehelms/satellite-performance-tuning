==============================
Satellite Configuration Tuning
==============================

Red Hat Satellite as a product comes with a number of components that communicate with each other to produce a final outcome. All these components can be tuned independently of each other to achieve the maximum possible performance for the scenario desired.

Tuned profile
=============

Red Hat Enterprise Linux 7 enables the tuned daemon by default during installation.  On bare-metal, it is recommended that Red Hat Satellite 6 and capsule servers run the ‘throughput-performance’ tuned profile. While, if virtualized, they should run the ‘virtual-guest’ profile. If it is not certain the system is currently running the correct profile, check with the ‘tuned-adm active’ command as shown above. More information about tuned is located in the Red Hat Enterprise Linux Performance Tuning Guide::

  # service tuned start
  # chkconfig tuned on
  RHEL 7 (bare-metal):
  # tuned-adm profile throughput-performance
  RHEL 7 (virtual machine) 
  # tuned-adm profile virtual-guest

Transparent Huge Pages is a memory management technique used by the Linux kernel which reduces the overhead of using Translation Lookaside Buffer (TLB) by using larger sized memory pages. Due to databases having Sparse Memory Access patterns instead of Contiguous Memory access patterns, database workloads often perform poorly when Transparent Huge Pages is enabled.
To improve performance of MongoDB, Red Hat recommends Transparent Huge Pages be disabled. For details on disabling Transparent Huge Pages, see `Red Hat Solution <https://access.redhat.com/solutions/1320153>`_.

Apache HTTPD Performance Tuning
===============================

Apache httpd forms a core part of the Satellite and acts as a web server for handling the requests that are being made through the Satellite Web UI or exposed APIs. To increase the concurrency of the operations, httpd forms the first point where tuning can help to boost the performance of the Satellite.

Configuring how many processes can be launched by Apache httpd
==============================================================

The version of Apache httpd that ships with Red Hat Satellite 6 by default uses prefork request handling mechanism. With the prefork model of handling the requests, httpd will launch a new process to handle the incoming connection by the client.

When the number of requests to the apache exceed the maximum number of child processes that can be launched to handle the incoming connections, an HTTP 503 Service Unavailable Error is raised by Apache.

Amidst httpd running out of processes to handle the incoming connections can also result in multiple component failure on the Satellite side due to the dependency of components like Pulp on the availability of httpd processes.

Based on your expected peak load, you might want to modify the configuration of apache prefork to enable it to handle more concurrent requests.

An example modification to the prefork configuration for a server which may desire to handle 150 concurrent content host registrations to Satellite may look like the configuration file example that follows (see how to use `custom-hiera.yaml` file; this will modify config file /etc/httpd/conf.modules.d/prefork.conf)::

  File: /etc/foreman-installer/custom-hiera.yaml
  apache::mod::prefork::serverlimit: 582
  apache::mod::prefork::maxclients: 582
  apache::mod::prefork::startservers: 10

In the above example, the ServerLimit parameter is set only to be able to raise MaxClients value.

The MaxClients (see MaxRequestWorker which is a new name in Apache docs) parameter is being used to set the maximum number of child processes that httpd can launch to handle the incoming requests.

The StartServers parameter defines how many server processes will be launched by default when the httpd process is started.


Increasing the MaxOpenFiles Limit
=================================

With the tuning in place, apache httpd can easily open a lot of file descriptors on the server which may exceed the default limit of most of the linux systems in place. To avoid any kind of issues that may arise as a result of exceeding max open files limit on the system, please create the following file and directory and set the contents of the file as specified in the below given example::

  File: /etc/systemd/system/httpd.service.d/limits.conf
  [Service]
  LimitNOFILE=640000

Once the file has been edited, the following commands need to be run to make the tunings come into effect::

  systemctl daemon-reload
  foreman-maintain service restart

Calculating the maximum open files limit for qdrouterd
======================================================

Calculate the limit for open files in qdrouterd using this formula: (Nx3) + 100, where N is the number of content hosts. Each content host may consume up to three file descriptors in the router, and 100 filedescriptors are required to run the router itself.

The following settings permit Satellite to scale up to 10,000 content hosts.

qdrouterd settings
==================

Add/Update qpid::router::open_file_limit  in custom-hiera.yaml as shown below::

  File: /etc/foreman-installer/custom-hiera.yaml
  qpid::router::open_file_limit: 150100

Note The change must be applied via::

  # satellite-installer
  # systemctl daemon-reload
  # foreman-maintain service restart

Calculating the maximum open files limit for qpidd
==================================================

Calculate the limit for open files in qpidd using this formula: (Nx4) + 500, where N is the number of content hosts. A single content host can consume up to four file descriptors and 500 file descriptors are required for the operations of Broker (a component of qpidd).

qpidd settings
==============

Add/Update qpid::open_file_limit in /etc/foreman-installer/custom-hiera.yaml as shown below::

  File: /etc/foreman-installer/custom-hiera.yaml
  qpid::open_file_limit: 65536

Note The change must be applied via::

  # satellite-installer
  # systemctl daemon-reload
  # foreman-maintain service restart

Maximum asynchronous input-output (AIO) requests
================================================

Increase the maximum number of allowable concurrent AIO requests by increasing the kernel parameter `fs.aio-max-nr`.

Edit configuration file `/etc/sysctl.conf`, setting the value of `fs.aio-max-nr` to the desired maximum.

  fs.aio-max-nr=23456

In this example, 23456 is the maximum number of allowable concurrent AIO requests.

This number should be bigger than 33 multiplied by the maximum number of the content hosts planned to be registered to Satellite. To apply the changes:

  sysctl -p

Rebooting the machine also ensures that this change is applied.

Storage Considerations
======================

Plan to have enough storage capacity for directory /var/lib/qpidd in advance when you are planning an installation that will use katello-agent extensively. In Red Hat Satellite 6, /var/lib/qpidd requires 2MB disk space per content host. See this `bug <https://bugzilla.redhat.com/show_bug.cgi?id=1366323>`_ for more details.

mgmt-pub-interval setting
=========================

You might see the following error in /var/log/journal in Red Hat Enterprise Linux 7::

  satellite.example.com qpidd[92464]: [Broker] error Channel exception: not-attached: Channel 2 is not attached(/builddir/build/BUILD/qpid-cpp-0.30/src/qpid/amqp_0_10/SessionHandler.cpp: 39)satellite.example.com    qpidd[92464]: [Protocol] error Connectionqpid.10.1.10.1:5671-10.1.10.1:53790 timed out: closing

This error message appears because qpid maintains management objects for queues, sessions, and connections and recycles them every ten seconds by default. The same object with the same ID is created, deleted, and created again. The old management object is not yet purged, which is why qpid throws this error. Here’s a workaround: lower the mgmt-pub-interval parameter from the default 10seconds to something lower. Add it to /etc/qpid/qpidd.conf and restart the qpidd service.  See also `Bug 1335694 <https://bugzilla.redhat.com/show_bug.cgi?id=1335694>`_ comment 7.

Puma Tuning
================

Puma is a ruby application server which is used for serving the Foreman related requests to the clients. 

For any Satellite configuration that is supposed to handle a large number of clients or frequent operations, it is important for the Puma to be tuned appropriately.

Threads min effects
--------------------
Less threads will lead to more memory usage for different scales on the Satellite server.

For example, we have compared these two setups:

+---------------------------------------+---------------------------------------+
| Satellite VM with 8 CPUs, 40 GB RAM   | Satellite VM with 8 CPUs, 40 GB RAM   |
+=======================================+=======================================+
| --foreman-service-puma-threads-min=0  | --foreman-service-puma-threads-min=16 |
+---------------------------------------+---------------------------------------+
| --foreman-service-puma-threads-max=16 | --foreman-service-puma-threads-max=16 |
+---------------------------------------+---------------------------------------+
| --foreman-service-puma-workers=2      | --foreman-service-puma-workers=2      |
+---------------------------------------+---------------------------------------+

When we tune the puma server with t_min=16 puma will consume about 12% less memory as compared to t_min=0.

Setting threads min, max & workers
-----------------------------------
More workers will allow for lower time to register hosts in parallel.

For example, we have compared these two setups:

+---------------------------------------+---------------------------------------+
| Satellite VM with 8 CPUs, 40 GB RAM   | Satellite VM with 8 CPUs, 40 GB RAM   |
+=======================================+=======================================+
| --foreman-service-puma-threads-min=16 | --foreman-service-puma-threads-min=8  |
+---------------------------------------+---------------------------------------+
| --foreman-service-puma-threads-max=16 | --foreman-service-puma-threads-max=8  |
+---------------------------------------+---------------------------------------+
| --foreman-service-puma-workers=2      | --foreman-service-puma-workers=4      |
+---------------------------------------+---------------------------------------+

In the second case with more workers but the same total number of threads, we have seen about 11% of speedup in highly concurrent registrations scenario. Moreover, adding more workers did not consume more cpu and memory but will get more performance.

Setting right number of workers for different number of CPUs
-------------------------------------------------------------
If you have enough CPUs, adding more workers adds more performance.

For example, we have compared Satellite setups with 8 and 16 CPUs.

+-----------------------------------------------+---------------------------------------------+
| Satellite VM with 8 CPUs, 40 GB RAM           | Satellite VM with 16 CPUs, 40 GB RAM        |
+===============================================+=============================================+
| --foreman-service-puma-threads-min=16         | --foreman-service-puma-threads-min=16       |
+-----------------------------------------------+---------------------------------------------+
| --foreman-service-puma-threads-max=16         | --foreman-service-puma-threads-max=16       |
+-----------------------------------------------+---------------------------------------------+
| --foreman-service-puma-workers=2,4,8 and 16   | --foreman-service-puma-workers=2,4,8 and 16 |
+-----------------------------------------------+---------------------------------------------+

In 8 CPUs setup, changing the number of workers from 2 to 16, improved concurrent registration time by 36%. In 16 CPU setup, the same change caused 55% improvement.

Adding more workers can also help with total registration concurrency Satellite can handle. In our measurements, setups with 2 workers were able to handle up to 480 concurrent registrations, but adding more workers improved the situation.


Dynflow Tuning
==============

Dynflow is the workflow management system and task orchestrator which is built as a plugin inside Foreman and is used to execute the different tasks of Satellite in an out-of-order execution manner. Under the conditions when there are a lot of clients checking in on Satellite and running a number of tasks, the Dynflow can take some help from an added tuning specifying how many executors can it launch.

In Satellite 6.8, Dynflow configuration has been changed as how this is configured entirely and we are working on a new dynflow configuration. Soon, will release a new version of the performance brief with new dynflow configuration. See these `examples <https://gist.github.com/adamruzicka/1991892ce22b18e030f9a4db95406319>`_ for more details. 

PostgreSQL Tuning
=================

PostgreSQL is the primary SQL based database that is used by Satellite for the storage of persistent context across a wide variety of tasks that Satellite does. The database sees an extensive usage is usually working on to provide the Satellite with the data which it needs for its smooth functioning. This makes PostgreSQL a heavily used process which if tuned can have a number of benefits on the overall operational response of Satellite.

The below set of tunings can be applied to PostgreSQL to improve its response times (see `how to use custom-hiera.yaml` file; this will modify /var/lib/pgsql/data/postgresql.conf file)::

  File: /etc/foreman-installer/custom-hiera.yaml
  postgresql::server::config_entries:
    max_connections: 1000
    shared_buffers: 2GB
    work_mem: 8MB
    autovacuum_vacuum_cost_limit: 2000

In the above tuning configuration, there are a certain set of keys which we have altered:

max_connections: The key defines the maximum number of connections that can be accepted by the PostgreSQL processes that are running. An optimal value for the parameter will be equal to the nearest multiple of 100 of the ServerLimit value of Apache httpd2 multiplied by 2. For example, if ServerLimit is set to 582, we can set the max_connections to 1000.

shared_buffers: The shared buffers define the memory used by all the active connections inside postgresql to store the data for the different database operations. An optimal value for this will vary between 2 GB to a maximum of 25% of your total system memory depending upon the frequency of the operations being conducted on Satellite.

work_mem: The work_mem is the memory that is allocated on per process basis for Postgresql and is used to store the intermediate results of the operations that are being performed by the process. Setting this value to 8 MB should be more than enough for most of the intensive operations on Satellite.

autovacuum_vacuum_cost_limit: The key defines the cost limit value for the vacuuming operation inside the autovacuum process to clean up the dead tuples inside the database relations. The cost limit defines the number of tuples that can be processed in a single run by the process. An optimal value for this is 2000 based on the general load that Satellite pushes on the PostgreSQL server process.

Note - With the upgrade to Postgres 12, ‘checkpoint_segments’ configuration is not supported. For more details, please refer to this `bugzilla <https://bugzilla.redhat.com/show_bug.cgi?id=1867311#c12>`_ . 

Benchmarking raw DB performance
===============================

To get a list of the top table sizes in disk space for both Candlepin and Foreman, check `postgres-size-report <https://github.com/RedHatSatellite/satellite-support/blob/master/postgres-size-report>`_ script in `satellite-support <https://github.com/RedHatSatellite/satellite-support>`_  git repository.

PGbench utility (note you may need to resize PostgreSQL data directory /var/lib/pgsql/ directory to 100GB or what does benchmark take to run) might be used to measure PostgreSQL performance on your system. Use yum install postgresql-contrib to install it. Some resources are:

 - https://github.com/RedHatSatellite/satellite-support

Choice of filesystem for PostgreSQL data directory might matter as well:

 - https://blog.pgaddict.com/posts/postgresql-performance-on-ext4-and-xfs

Note:

 - Never do any testing on production system and without valid backup.
 - Before you start testing, see how big the database files are. Testing with a really small database would not produce any meaningful results. E.g. if the DB is only 20G and the buffer pool is 32G, it won't show problems with large number of connections because the data will be completely buffered.

MongoDb Tuning
==============

Under certain circumstances, mongod consumes randomly high memory (up to 1/2 of all RAM) and this aggressive memory usage limits other processes or can cause OOM killer to kill mongod. In order to overcome this situation, tune the cache size by referring the following steps:

**1.** Update custom-hiera.yaml:

Edit /etc/foreman-installer/custom-hiera.yaml and add the entry below inserting the value that is 20% of the physical RAM while keeping in mind the `guidlines <https://access.redhat.com/documentation/en-us/red_hat_satellite/6.9-beta/html/installing_satellite_server_from_a_connected_network/preparing-environment-for-satellite-installation#storage-guidelines_satellite>`_ in this case, approximately 6GB for a 32GB server. Please note the formatting of the second line and the indent::

  mongodb::server::config_data:
   storage.wiredTiger.engineConfig.cacheSizeGB: 6

**2.** Run installer to apply changes::

  # satellite-installer


For more details, please refer to this Kbase `article <https://access.redhat.com/solutions/4505561>`_. 


Benchmarking raw performance
============================

To get a size report of MongoDB, use `mongo-size-report <https://github.com/RedHatSatellite/satellite-support/blob/master/mongo-size-report>`_ from `satellite-support <https://github.com/RedHatSatellite/satellite-support/>`_  repository.

Utility used for checking IO speed specific to MongoDB:

 - https://www.mongodb.com/blog/post/checking-disk-performance-with-the-mongoperf

For MongoDB benchmark meant to run on (stage) Satellite installs, check `mongo-benchmark <https://github.com/RedHatSatellite/satellite-support/blob/master/mongo-benchmark>`_ tool in `satellite-support <https://github.com/RedHatSatellite/satellite-support>`_ git repository.

Depending on a disk drive type, file system choice (ext4 or xfs) for MongoDB storage directory might be important:

 - https://scalegrid.io/blog/xfs-vs-ext4-comparing-mongodb-performance-on-aws-ec2/

Note:

 - Never do any testing on production system and without valid backup.
