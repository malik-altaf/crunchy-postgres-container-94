Crunchy PostgreSQL 9.4.5
==========================

This project includes a Dockerfile that lets you build
a PostgreSQL 9.4.5 Docker image.  The image by default
is built on a RHEL 7.2 64 bit base, but can also be built
on a centos 7 64 bit base.  NOTICE, to build the RHEL 7 
version of this container, you need to build the Docker
container on a licensed RHEL 7 host!

Installation
------------

Docker image builds are performed by issuing the make command:
~~~~~~~~~~~~~~~~~~~~~
make
~~~~~~~~~~~~~~~~~~~~~

This will build three Docker images:

 * crunchy-pg

For most of the examples, I use the psql command as a postgres client.  To install PostgreSQL locally, run the following:
~~~~~~~~~~~~~~~~~~~~~
sudo rpm -Uvh http://yum.postgresql.org/9.4/redhat/rhel-7-x86_64/pgdg-redhat94-9.4-1.noarch.rpm
sudo yum -y install postgresql94-server postgresql94
~~~~~~~~~~~~~~~~~~~~~

The installation of postgres locally is also required to create a postgres user and group which are
used by the examples to set the correct file permissions on local files used by the containers.

crunchy-pg Configuration Options
--------------------------------

You can adjust the following Postgres configuration parameters
by setting environment variables:
~~~~
MAX_CONNECTIONS - defaults to 100
SHARED_BUFFERS - defaults to 128MB
TEMP_BUFFERS - defaults to 8MB
WORK_MEM - defaults to 4MB
MAX_WAL_SENDERS - defaults to 6
~~~~

You have the ability to override the pg_hba.conf and postgresql.conf
files used by the container.  To enable this, you create a 
directory to hold your own copy of these configuration files.

Then you mount that directory into the container using the /pgconf
volume mount as follows:

~~~~~~~
-v $YOURDIRECTORY:/pgconf
~~~~~~~

Inside YOURDIRECTORY would be your pg_hba.conf and postgresql.conf
files.  These files are not manipulated or changed by the container
start scripts.

### Example 1 - Running crunchy-pg without using Openshift

Run the container with this command:
~~~~~~~~~~~~~~~~~~~~~
./run-pg-master.sh
~~~~~~~~~~~~~~~~~~~~~

This script will do the following:
 * start up a container named master
 * create a data directory for the database in /tmp/master-data
 * initialize the database using the passed in environment variables, building a user, database, schema, and table based on the environment variable values.
 * maps the PostgreSQL port of 5432 in the container to your local host port of 12000.

The container creates a default database called 'testdb', a default
user called 'testuser' with a default password of 'testpsw', you can
use this to connect from your local host as follows:
~~~~~~~~~~~~~~~~~~~~~
psql -h localhost -p 12000 -U testuser -W testdb
~~~~~~~~~~~~~~~~~~~~~

To shut down the instance, run the following commands:

~~~~~~~~~~~~~~~~~~~~~
docker stop master
~~~~~~~~~~~~~~~~~~~~~
	
To start the instance, run the following commands:

~~~~~~~~~~~~~~~~~~~~~
docker start master
~~~~~~~~~~~~~~~~~~~~~
	
### Example 2 - Running PostgreSQL in Master-Slave Configuration

The container can be passed environment variables that will cause
it to assume a PostgreSQL replication configuration with 
a master instance streaming to a slave replica instance.

The following env variables are specified for this configuration option:
~~~~~~~~~~~~~~~~~~~
PG_MASTER_USER - The username used for master-slave replication value=master
PG_MASTER_PASSWORD - The password for the PG master user
PG_USER - The username that clients will use to connect to PG server value=user
PG_PASSWORD  - The password for the PG master user
PG_DATABASE - The name of the database that will be created value=userdb
PG_ROOT_PASSWORD - The password for the PG admin
~~~~~~~~~~~~~~~~~~~

This examples assumes you have run Example 1, and that the master
container is running.

For running the master-slave configuration , you can run the following scripts:
~~~~~~
run-pg-slave.sh
~~~~~~

You can verify that replication is working by the following commands:
~~~
psql -h 0.0.0.0 -p 12001 -U postgres postgres

~~~

## Openshift Examples

### Install Openshift Origin

Binary releases of Openshift Origin are available here:
https://github.com/openshift/origin/releases

Openshift install steps:
 * download the binary release
 * untar the release to your local host
 * run as follows:
~~~
sudo ./openshift start
~~~
 * set the permissions on the admin kubeconfig file to allow changes
~~~
sudo chmod 666 ./openshift.local.config/master/admin.kubeconfig
~~~
 * add the following to your bash shell environment to allow
 you access to the openshift admin account:
~~~
export KUBECONFIG="$(pwd)"/openshift.local.config/master/admin.kubeconfig
export CURL_CA_BUNDLE="$(pwd)"/openshift.local.config/master/ca.crt
~~~
 * edit the restricted settings, you will need to change the runAsUser type value to RunAsAny to allow the container to
run as the postgres user.
~~~
oc login -u system:admin
oc edit scc restricted --config=./openshift.local.config/master/admin.kubeconfig
~~~
 * login in as test user and create a project called pgproject
~~~
oc login -u test
oc new-project pgproject
~~~

Openshift starts an internal DNS server when it starts, and it
registers DNS names for each service that you create.  To
refer to this DNS server, you adjust your /etc/resolv.conf
file to include the Openshift DNS server IP as your primary
name server, also you adjust the searchpath if you
dont want to type '<<<projectname>>.svc.cluster.local when
you refer to the various Openshift services you create.  Here
is an example /etc/resolv.conf that uses an Openshift project
name of 'pgproject':

~~~
search pgproject.svc.cluster.local crunchy.lab
nameserver 192.168.122.71
nameserver 192.168.122.1
~~~

This example used an Openshift project name of 'pgproject'.  The
project name is used as part of the DNS names set by Openshift
for Services.  

### Prerequisites

NFS is able to run in selinux Enforcing mode if you 
following the instructions here:

https://github.com/openshift/origin/tree/master/examples/wordpress

Other information on how to install and configure an NFS share is located
here:

http://www.itzgeek.com/how-tos/linux/centos-how-tos/how-to-setup-nfs-server-on-centos-7-rhel-7-fedora-22.html

Examples of Openshift NFS can be found here:

https://github.com/openshift/origin/tree/master/examples/wordpress/nfs

The examples specify a test NFS server running at IP address 192.168.0.103

On that server, the /etc/exports file looks like this:

~~~
/jeffnfs * (rw,sync)
~~~

if you are running your client on a VM, you will need to
add 'insecure' to the exportfs file on the NFS server, this is because
of the way port translation is done between the VM host and the VM instance.

see this for more details:

http://serverfault.com/questions/107546/mount-nfs-access-denied-by-server-while-mounting


### Openshift Example 1 - master.json

This openshift template will create a single master PostgreSQL instance.


Running the example:

~~~~~~~~~~~~~~~~
oc create -f master.json | oc create -f -
~~~~~~~~~~~~~~~~

You can see what passwords were generated by running this command:

~~~
oc describe pod pg-master | grep PG
~~~

You can run the following command to test the database, entering
the value of PG_PASSWORD from above for the password when prompted:

~~~~~~~~~~~~~~
psql -h pg-master.pgproject.svc.cluster.local -U testuser userdb
~~~~~~~~~~~~~~

### Openshift Example 2 - slave.json

This openshift template will create a single slave PostgreSQL instance
that will connect to the master created in Example 1.

You will need to edit the slave.json file and enter the PG_MASTER_PASSWORD
value generated in Example 1 as found in the master environment variables.


Running the example:

~~~~~~~~~~~~~~~~
oc create -f slave.json | oc create -f -
~~~~~~~~~~~~~~~~

You can run the following command to test the database, entering
the value of PG_PASSWORD from above for the password when prompted:

~~~~~~~~~~~~~~
psql -c 'select * from pg_stat_replication' -h pg-master.pgproject.svc.cluster.local -U master userdb
psql -c 'create table foo (id int)' -h pg-master.pgproject.svc.cluster.local -U master userdb
psql -c 'insert into foo values (123)' -h pg-master.pgproject.svc.cluster.local -U master userdb
psql -c 'table foo' -h pg-slave.pgproject.svc.cluster.local -U master userdb
~~~~~~~~~~~~~~

If replication is working, you should see a row returned in the
pg_stat_replication table query and also a row returned from
the pg-slave query.

 
### Openshift Example 4 - master-slave-rc.json

This example is similar to the previous examples but
builds a master pod, and a single slave that can be scaled up
using a replication controller.   The master is implemented as
a single pod since it can not be scaled like read-only slaves.

Running the example:

~~~~~~~~~~~~~~~~
oc create -f master-slave-rc.json | oc create -f -
~~~~~~~~~~~~~~~~

Connect to the PostgreSQL instances with the following:

~~~~~~~~~~~~~~
psql -h pg-master-rc.pgproject.svc.cluster.local -U testuser userdb
psql -h pg-slave-rc.pgproject.svc.cluster.local -U testuser userdb
~~~~~~~~~~~~~~

Here is an example of increasing or scaling up the Postgres 'slave' pods to 2:

~~~~~~~~~~
oc scale rc pg-slave-rc-1 --replicas=2
~~~~~~~~~~

Enter the following commands to verify the PostgreSQL replication is working.

~~~
psql -c 'table pg_stat_replication' -h pg-master-rc.pgproject.svc.cluster.local -U master postgres
psql -h pg-slave-rc.pgproject.svc.cluster.local -U master postgres
~~~

You can see that the slave service is load balancing between
multiple slaves by running a command as follows, run the command
multiple times and the ip address should alternate between
the slaves:

~~~
psql -c 'select inet_server_addr()' -h pg-slave-rc -U master postgres
~~


### Openshift Example 5 - NFS Example

I have provided an example of using NFS for the postgres data volume.
On my test nfs server, I had to set the exports file entry as follows:
~~~
/jeffnfs * (rw,insecure,sync)
~~~

First, you can only create persistent volumes as a cluster admin, you can
login in as the admin user as follows:

~~~
oc login -u system:admin
~~~

To run it, you would execute the following as the openshift administrator:

~~~~~~~~~~~~~~~
oc create -f master-nfs-pv.json
~~~~~~~~~~~~~~~

Then as the normal openshift user account, create the Persistence Volume
Claim and database pod as follows:
~~~~~~~~~~~~~~~
oc create -f master-nfs-pvc.json
oc process -f master-nfs.json | oc create -f -
~~~~~~~~~~~~~~~

This will create a single master postgres pod that is using 
an NFS volume to store the postgres data files.

### Openshift Example 6 - Failover Example

An example of performing a database failover is described
in the following steps:

 * create a master and slave replication using master-slave-rc.json
~~~
oc process -f master-slave-rc.json | oc create -f -
~~~
 * scale up the number of slaves to 2
~~~
oc scale rc pg-slave-rc-1 --replicas=2
~~~
 * delete the master pod
~~~
oc delete pod pg-master-rc
~~~
 * exec into a slave and create a trigger file to being
the recovery process, effectively turning the slave into a master
~~~
oc exec -it pg-slave-rc-1-lt5a5
touch /tmp/pg-failover-trigger
~~~
 * change the label on the slave to pg-master-rc instead of pg-slave-rc
~~~
oc edit pod/pg-slave-rc-1-lt5a5
original line: labels/name: pg-slave-rc
updated line: labels/name: pg-master-rc
~~~
or alternatively:
~~~
oc label --overwrite=true pod pg-slave-rc-1-lt5a5 name=pg-master-rc
~~~

You can test the failover by creating some data on the master
and then test to see if the slaves have the replicated data.

~~~
psql -c 'create table foo (id int)' -U master -h pg-master-rc postgres
psql -c 'table foo' -U master -h pg-slave-rc postgres
~~~

After a failover, you would most likely want to create a database
backup and be prepared to recreate your cluster from that backup.

## Openshift Tips

### Tip 1: Finding the Postgresql Passwords

The passwords used for the PostgreSQL user accounts are generated
by the Openshift 'process' command.  To inspect what value was
supplied, you can inspect the master pod as follows:

~~~~~~~~~~~~~~~
oc get pod pg-master-rc-1-n5z8r -o json
~~~~~~~~~~~~~~~

Look for the values of the environment variables:
- PG_USER
- PG_PASSWORD
- PG_DATABASE


