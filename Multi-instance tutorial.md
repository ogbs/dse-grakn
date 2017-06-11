# DSE 5.0 Multi-Instance Demo Tutorial

The new Multi-Instance feature released with DSE 5.0 allows for the simple deployment of multiple DSE instances on a single machine.

This helps customer ensure that large hardware resources are effectively utilized for database applications, and making more efficient use of hardware can help reduce overall project costs.

DataStax Multi-Instance documentation can be found here: https://docs.datastax.com/en/latest-dse/datastax_enterprise/multiInstance/configMultiInstance.html


## Build a DSE Node
You can use a bare machine, an empty VM, a docker image, whatever. I recommend at least a couple of CPU's and 6GB+ prefereably 8GB.

Most leading flavours of Linux supported. Here I'm using Ubuntu Precise64 that comes with the build I'm using.

I'm using a Virtualbox VM created using Vagrant that provisions a single large node designed to support multiple instances of DataStax Enterprise. The repo for that can be found here: https://github.com/simonambridge/vagrant-DSE-multi-instance

>This Vagrant build is based on Joel Jacobson's great multi-node vagrant build that cerates a three node cluster with an opscentre VM. You'll find it described here: https://github.com/joeljacobson/vagrant-ansible-cassandra

Whatever you're using, start your VM or environment and move on to the next section for configuration details.

If you're using vagrant, after you've built the machine or vagrant virtual box, if it isn't running, start it with:

```
vagrant up dse-node
```
You can check what VM's are available to vagrant:
```
vagrant global-status
```


## Configure The VM for DSE

Now log into the DSE node. For the vagrant build use:
```
vagrant ssh dse-node
```


### Update ulimits
You first need to update the ulimits so that your vagrant user (or whatever OS cassandra username you'll use) can start DSE.
There is a separate conf file to set ulimits for the cassandra user
```
sudo vi /etc/security/limits.d/cassandra.conf
```
Add or update the entries in there to look like this:
```
vagrant - memlock unlimited
vagrant - nofile 100000
vagrant - nproc 32768
vagrant - as unlimited
```
Now log out and ssh back into the box as the vagrant user in order to pick up the new ulimits.


### Create Virtual IP Addresses For The Instances
Back in the vagrant VM, we need to create some IP aliases for our new instances - use the ```ifconfig``` command:
```
ifconfig lo:0 127.0.0.2 netmask 255.0.0.0 up
ifconfig lo:1 127.0.0.3 netmask 255.0.0.0 up
ifconfig lo:2 127.0.0.4 netmask 255.0.0.0 up
```


### Create Aliases for the Virtual IPs
Update ```/etc/hosts``` to make it easier with aliases:
```
127.0.0.2 node1 dse-node1
127.0.0.3 node2 dse-node2
127.0.0.4 node3 dse-node3
```


## Add Node 1

### Use the ````add-node``` utility to add a multi-instance node

You can call your node what you like but it will be prefixed automatically with ```dse-```. So ```node1``` will be ```dse-node1``` etc.

We don't have a valid seed address here as we're creating the first node.

> NB Do not attempt to add a node to the default dse node service that gets created when you install DSE. DSE Multi-Instance maintains a naming nomenclature for the nodes it creates and manages, and the dse service doesnt fit in. Keeping it running will just confuse things when you have nodes managed by multi-instance. Shut it down and keep it to play with when youre not playing with multi-instance. YMMV

So, now we know that we only need the installation of DSE to give us the binaries, config templates and filiesystem layout. We don't need the service that is pre-configured for a stand-alone instance.

Lets add the first node. We'll call it node1 (so it will be called dse-node1), create a cluster called Cassandra, using the first of the IP aliases that we created. We don't have a seed to talk to so we use ourself as a seed:
```
sudo dse add-node --node-id node1 --cluster Cassandra --listen-address 127.0.0.2 --rpc-address 127.0.0.2 --seeds 127.0.0.2
Installing and configuring from /usr/share/dse/templates/

+ Setting up node dse-node1...
  - Copying configs
      - Setting the cluster name.
      - Setting up JMX port
      - Setting up directories
Warning: Spark shuffle service port not set. Spark nodes will use the default binding options.
Done.
```

You can specify the rack ID e.g. if you are setting up nodes on a different machine (default is rack 1):
```
--rack=rack_name
```


The template filesystem structure for this new node will now have been created. You'll find configuration files in:

```
sudo ls /etc/dse-node1
byoh	     cassandra	 dse-node1.init  dse.yaml  hadoop	   hive    pig	 spark	tomcat
byoh-env.sh  dse-env.sh  dserc-env.sh	 graph	   hadoop2-client  mahout  solr  sqoop
```

Here you set whether you want it to be a Cassandra, Spark, Solr node etc:

> ignore this if you just want Cassandra

```
sudo vi /etc/default/dse-node1
```

Data lives here:
```
sudo ls /var/lib/dse-node1
commitlog  data  hints	saved_caches  spark
```

Logs live here:
```
sudo ls /var/log/dse-node1
audit  debug.log  gremlin.log  output.log  spark  system.log
```

Although ```add-node``` told you that it was setting up the JMX port, I found that it uses 7199 each time it runs (e.g. it doesnt appear to check if the port is in use) so you need to set the port yourself. 

I set the port on the first node just so that it doesnt conflict with the port used by the default dse service that I might want to start one day.

### Set the Node 1 JMX port ```JMX_PORT="7299"```
The nodetool utility communicates through JMX on port 7199. We need to change it for our instance. 

>As its running on a different host IP, you should theoretically be able to use 7199, but I found that nodetool didn't recognise the aliases but would bind with just the JMX address e.g. ```nodetool -p 7299```)

Edit the cassandra-env.sh file for this node.

Search for 7199 and change it to 7299

```
sudo vi /etc/dse-node1/cassandra/cassandra-env.sh

JMX_PORT="7299"
JVM_OPTS="$JVM_OPTS -Djava.rmi.server.hostname=127.0.0.2"
```

### Start The Node 1 Service
```
sudo service dse-node1 start
 * Starting DSE daemons (dse-node1) dse-node1
DSE daemons (dse-node1) starting with only Cassandra enabled (edit /etc/default/dse-node1 to enable other features)
```

Check the log file to make sure everything's rosy:
```
sudo tail -100 /var/log/dse-node1/system.log
```

If all goes to plan you should see:
```
INFO  [main] 2016-07-13 05:46:18,137  ThriftServer.java:119 - Binding thrift service to /127.0.0.2:9160
INFO  [Thread-3] 2016-07-13 05:46:18,145  ThriftServer.java:136 - Listening for thrift clients...
INFO  [main] 2016-07-13 05:46:18,145  DseDaemon.java:827 - DSE startup complete.
```

We can check the cluster with the new multi-instance support - use the node name:
```
sudo dse dse-node1 dsetool ring
Address          DC                   Rack         Workload             Graph  Status  State    Load             Owns                 Token                                        Health [0,1]
127.0.0.2        Cassandra            rack1        Cassandra            no     Up      Normal   133.53 KB        ?                    -9098225573757054999                         0.00
```

We can access via cqlsh:
```
$ cqlsh 127.0.0.2
Connected to Cassandra at 127.0.0.2:9042.
[cqlsh 5.0.1 | Cassandra 3.0.7.1159 | DSE 5.0.1 | CQL spec 3.4.0 | Native protocol v4]
Use HELP for help.
cqlsh> exit
```

## Add Node 2

Same command as before, same cluster name, this time with the next available virtual IP address.
This time we  point to the first node that we created to use it as a seed node

```
sudo dse add-node --node-id node2 --cluster Cassandra --listen-address 127.0.0.3 --rpc-address 127.0.0.3 --seeds 127.0.0.2
Installing and configuring from /usr/share/dse/templates/

+ Setting up node dse-node2...
  - Copying configs
      - Setting the cluster name.
      - Setting up JMX port
      - Setting up directories
Warning: Spark shuffle service port not set. Spark nodes will use the default binding options.
Done.
```

### Set the Node 2 JMX port ```JMX_PORT="7399"```

Search for 7199 and change it to 7399, and set the JMX hostname to the IP alias for this instance:

```
sudo vi /etc/dse-node2/cassandra/cassandra-env.sh
JMX_PORT="7399"
JVM_OPTS="$JVM_OPTS -Djava.rmi.server.hostname=127.0.0.3"
```

### Start The Node 2 Service

```
sudo service dse-node2 start
 * Starting DSE daemons (dse-node2) dse-node2
DSE daemons (dse-node2) starting with only Cassandra enabled (edit /etc/default/dse-node2 to enable other features)
```

Check it:
```
sudo tail -100 /var/log/dse-node2/system.log
```

How does my cluster look? Notice how the machine ID is displayed when you have more than one node in a cluster - so you know where your nodes are running:
```
sudo dse dse-node1 dsetool ring
Server ID          Address          DC                   Rack         Workload             Graph  Status  State    Load             Owns                 Token                                        Health [0,1]
                                                                                                                                                         -9024954150119957189
08-00-27-88-0C-A6  127.0.0.2        Cassandra            rack1        Cassandra            no     Up      Normal   133.53 KB        ?                    -9098225573757054999                         0.10
08-00-27-88-0C-A6  127.0.0.3        Cassandra            rack1        Cassandra            no     Up      Normal   116.85 KB        ?                    -9024954150119957189                         0.10
```

Access from cqlsh:
```
cqlsh 127.0.0.3
Connected to Cassandra at 127.0.0.3:9042.
[cqlsh 5.0.1 | Cassandra 3.0.7.1159 | DSE 5.0.1 | CQL spec 3.4.0 | Native protocol v4]
Use HELP for help.
cqlsh> exit
```

We can have a peek and see the cluster name we specified - the data cantre is also called Cassandra:
```
cqlsh> select cluster_name from system.local;

 cluster_name
--------------
   Cassandra
(1 rows)

cqlsh> select data_center from system.local;

 data_center
-------------
   Cassandra
(1 rows)
```
The gossip info shows we are up on our IP aliases:
```
sudo nodetool -p 7299 gossipinfo
/127.0.0.2
  generation:1469729834
  heartbeat:312526
  STATUS:18:NORMAL,-9098225573757054999
  LOAD:312403:2.6772283E7
  SCHEMA:35:335c1225-bfd5-3a7a-83d4-8cafde713850
  DC:54:Cassandra
  RACK:12:rack1
  RELEASE_VERSION:4:3.0.7.1159
  RPC_ADDRESS:3:127.0.0.2
  X_11_PADDING:264567:{"dse_version":"5.0.1","workload":"Cassandra","server_id":"08-00-27-88-0C-A6","graph":false,"active":"true","health":0.9}
  SEVERITY:312528:0.0
  NET_VERSION:1:10
  HOST_ID:2:52840937-5fe7-4586-85c3-51077a4dfc35
  RPC_READY:59:true
  TOKENS:17:<hidden>
/127.0.0.3
  generation:1469729819
  heartbeat:312568
  STATUS:20:NORMAL,-9024954150119957189
  LOAD:312399:2.423628E7
  SCHEMA:79:335c1225-bfd5-3a7a-83d4-8cafde713850
  DC:67:Cassandra
  RACK:12:rack1
  RELEASE_VERSION:4:3.0.7.1159
  RPC_ADDRESS:3:127.0.0.3
  X_11_PADDING:264770:{"dse_version":"5.0.1","workload":"Cassandra","server_id":"08-00-27-88-0C-A6","graph":false,"active":"true","health":0.9}
  SEVERITY:312567:0.0
  NET_VERSION:1:10
  HOST_ID:2:4a1985eb-075e-43da-8a9f-d13717d371ff
  RPC_READY:81:true
  TOKENS:19:<hidden>
```

## Add Node 3 - Standalone

Now we're going to create a new multi-instance managed node, but this time in a different cluster.

Same command as before, different cluster name, different virtual IP address, and we don't have a seed address.

```
sudo dse add-node --node-id node3 --cluster NewCluster --listen-address 127.0.0.4 --rpc-address 127.0.0.4 --seeds 127.0.0.4
Installing and configuring from /usr/share/dse/templates/

+ Setting up node dse-node3...
  - Copying configs
      - Setting the cluster name.
      - Setting up JMX port
      - Setting up directories
Warning: Spark shuffle service port not set. Spark nodes will use the default binding options.
Done.
```

Set the jmx port JMX_PORT="7499"

Search for 7199 and change it to 7499, and set the JMX hostname to the IP alias for this instance:

```
sudo vi /etc/dse-node3/cassandra/cassandra-env.sh
JMX_PORT="7499"
JVM_OPTS="$JVM_OPTS -Djava.rmi.server.hostname=127.0.0.4"
```

Check your different cluster name ```cluster_name: NewCluster```
```
sudo vi /etc/dse-node3/cassandra/cassandra.yaml
```

### Start The Node 3 Service
```
sudo service dse-node3 start
 * Starting DSE daemons (dse-node3) dse-node3
DSE daemons (dse-node3) starting with only Cassandra enabled (edit /etc/default/dse-node3 to enable other features)
```

And again:
```
sudo tail -100 /var/log/dse-node3/system.log
```
We want to see:
```
INFO  [main] 2016-07-13 13:07:29,996  ThriftServer.java:119 - Binding thrift service to /127.0.0.4:9160
INFO  [Thread-3] 2016-07-13 13:07:30,005  ThriftServer.java:136 - Listening for thrift clients...
INFO  [main] 2016-07-13 13:07:30,005  DseDaemon.java:827 - DSE startup complete.
```
We can access with cqlsh:
```
cqlsh 127.0.0.4
Connected to NewCluster at 127.0.0.4:9042.
[cqlsh 5.0.1 | Cassandra 3.0.7.1159 | DSE 5.0.1 | CQL spec 3.4.0 | Native protocol v4]
Use HELP for help.
cqlsh> exit
```

We can look at the first cluster we created:
```
sudo dse dse-node2 dsetool ring
Server ID          Address          DC                   Rack         Workload             Graph  Status  State    Load             Owns                 Token                                        Health [0,1]
                                                                                                                                                         -9024954150119957189
08-00-27-88-0C-A6  127.0.0.2        Cassandra            rack1        Cassandra            no     Up      Normal   133.53 KB        ?                    -9098225573757054999                         0.20
08-00-27-88-0C-A6  127.0.0.3        Cassandra            rack1        Cassandra            no     Up      Normal   116.99 KB        ?                    -9024954150119957189                         0.20
Note: you must specify a keyspace to get ownership information.
```

We can look at the second cluster:
```
sudo dse dse-node3 dsetool ring
Address          DC                   Rack         Workload             Graph  Status  State    Load             Owns                 Token                                        Health [0,1]
127.0.0.4        Cassandra            rack1        Cassandra            no     Up      Normal   88.15 KB         ?                    -2000014393878253047                         0.00
Note: you must specify a keyspace to get ownership information.
```

We can poke around a bit in the second cluster and see that it has a different cluster name:
```
cqlsh> select cluster_name from system.local;

 cluster_name
--------------
   NewCluster
(1 rows)

cqlsh> select data_center from system.local;

 data_center
-------------
   Cassandra
(1 rows)
```


## Connect with JMX

We can check the JMX parameters in use in the process JVM Arguments:
```"-Djava.rmi.server.hostname=127.0.0.2, -Dcassandra.jmx.local.port=7299,"```

Use the JMX port with nodetool to connect:

```
nodetool -p 7299 status
Datacenter: Cassandra
=====================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address    Load       Tokens       Owns    Host ID                               Rack
UN  127.0.0.2  18.29 MB   1            ?       52840937-5fe7-4586-85c3-51077a4dfc35  rack1
UN  127.0.0.3  15.38 MB   1            ?       4a1985eb-075e-43da-8a9f-d13717d371ff  rack1

Note: Non-system keyspaces don't have the same replication settings, effective ownership information is meaningless
```
Check that JMX is up for node1:
```
netstat -tulpn | grep 7299
(No info could be read for "-p": geteuid()=1000 but you should be root.)
tcp        0      0 127.0.0.1:7299          0.0.0.0:*               LISTEN      -
```

And for node2:
```
netstat -tulpn | grep 7399
(No info could be read for "-p": geteuid()=1000 but you should be root.)
tcp        0      0 127.0.0.1:7399          0.0.0.0:*               LISTEN
```

## Things go wrong sometimes - if you need to remove a node....

Use the ```remove-node``` tool:
```
$ sudo dse remove-node node1 --purge
##############################
#
# WARNING
# You're trying to remove node dse-node1
# This means that all configuration files for dse-node1 will be deleted
#
##############################

Do you wish to continue?
1) Yes
2) No
#? Yes
#? 1
 * Stopping DSE daemons (dse-node1) dse-node1                                                                                                                                               [ OK ]
Deleting /etc/dse/serverconfig/dse-node1
Deleting /etc/dse-node1
Deleting /etc/init.d/dse-node1
Deleting /etc/default/dse-node1
Deleting /var/run/dse-node1
```
Use the purge option to clean up data and logs or do it manually:
```
sudo rm -rf /var/lib/dse-node3
sudo rm -rf /var/log/dse-node3
```

## Multi-instance Ops Manager

The new Multi-Instance feature released with DSE 5.0 allows for the simple deployment of multiple DSE instances on a single machine.

For the first part in this series on Multi-Instance look here.

DataStax Multi-Instance documentation can be found here

### Install DataStax OpsCenter

```
curl -L https://debian.datastax.com/debian/repo_key | sudo apt-key add -
sudo apt-get update
```
Let's check the nodes we have installed from the previous lesson:

```
sudo dse list-nodes
dse  dse-node1  dse-node2  dse-node3
```

Thus:
```
dse-node1 is on 127.0.0.2 JMX=7299 cluster
dse-node2 is on 127.0.0.3 JMX=7399 cluster
dse-node3 is on 127.0.0.4 JMX=7499 single 
```

### Install OpsCenter

Use apt-get per `sudo apt-get install opscenter`
You should get some output like this: 

```
writing new private key to '/var/lib/opscenter/ssl/opscenter.key'
-----
MAC verified OK
Certificate was added to keystore
Warning: Overwriting existing alias agent_key in destination keystore
[Storing /var/lib/opscenter/ssl/agentKeyStore.p12]
MAC verified OK
```

Now start OpsCenter: `sudo service opscenterd start`
Check its working correctly:

```
sudo tail -100 /var/log/opscenter/startup.log
sudo tail -100 /var/log/opscenter/opscenterd.log
```

Check the web service is working:

http://192.168.56.10:8888/opscenter/index.html
You'll be prompted to manage an existing cluster:

You can set up nodes at 127.0.0.2/7299 and 127.0.0.3/7399

### Tarball agent install - dse-node1

```
sudo mkdir -p /usr/share/datastax-agent/dse-node1
cd /usr/share/datastax-agent/dse-node1

sudo curl --user jonathan_reiter@mcafee.com:'1 Cool Dude' -L http://downloads.datastax.com/enterprise/datastax-agent-6.1.tar.gz | sudo tar xz
```

What have we got?

```
ls /usr/share/datastax-agent/dse-node1
datastax-agent-6.1.0
cd /usr/share/datastax-agent/dse-node1/datastax-agent-6.1.0/conf
```

Edit the address.yaml to set the OpsCenter parameters - the stomp interface points to the node with OpsCenter per `vi ./conf/address.yaml`

```
stomp_interface: 127.0.0.1
agent_rpc_interface: 127.0.0.2
jmx_port: 7299
```

Start agent as root per `sudo bin/datastax-agent`
You should get output like this:

```
  INFO [async-dispatch-2] 2016-07-25 15:17:38,104 Starting RollupComponent at 200/second [<127]
  INFO [async-dispatch-2] 2016-07-25 15:17:38,130 Attempting to load stored metric values.
  INFO [async-dispatch-2] 2016-07-25 15:17:38,138 Completed loading 0 stored rollup states
  INFO [async-dispatch-2] 2016-07-25 15:17:38,140 Starting OSStatCollection (Linux)
  INFO [async-dispatch-2] 2016-07-25 15:17:38,152 Starting Performance Service
  INFO [async-dispatch-2] 2016-07-25 15:17:38,165 Starting JMXMetricComponent
  INFO [async-dispatch-2] 2016-07-25 15:17:38,166 Starting Cassandra JMX metric collectors
  INFO [async-dispatch-2] 2016-07-25 15:17:38,180 Finished starting system.
  INFO [qtp1424650534-55] 2016-07-25 15:17:55,489 HTTP: :get /connection-status {} - 200
```

After a few seconds you should see your agent on the cluster you defined earlier.

http://192.168.56.10:8888/opscenter/index.html

### Tarball agent install - dse-node2

Create a directory for the agent

```
sudo mkdir -p /usr/share/datastax-agent/dse-node2
cd /usr/share/datastax-agent/dse-node2
```

Download and unpack the Agent Install File

`sudo curl --user jonathan_reiter@mcafee.com:'1 Cool Dude' -L http://downloads.datastax.com/enterprise/datastax-agent-6.1.tar.gz | sudo tar xz`

Output:
```
ls /usr/share/datastax-agent/dse-node2
datastax-agent-6.1.0
```

`cd /usr/share/datastax-agent/dse-node2/datastax-agent-6.1.0/conf`
Edit the address.yaml file for node2 - the stomp interface points to the node with OpsCenter per `vi ./conf/address.yaml`

```
stomp_interface: 127.0.0.1
agent_rpc_interface: 127.0.0.3
jmx_port: 7399
```

Start the second agent as root per `sudo bin/datastax-agent` 

Again, look for success:
```
  INFO [async-dispatch-2] 2016-07-28 23:25:15,340 Starting Cassandra JMX metric collectors
  INFO [async-dispatch-2] 2016-07-28 23:25:15,409 Finished starting system.
  INFO [qtp1926072223-57] 2016-07-28 23:25:33,542 HTTP: :get /connection-status {} - 200
  INFO [qtp1926072223-55] 2016-07-28 23:26:33,460 HTTP: :get /connection-status {} - 200  
```
And check OpsCenter:

  http://192.168.56.10:8888/opscenter/index.html

# Adding Notional Data

## Create The Schema On My Two Node Cluster

Create the keyspace:

`cqlsh dse-node1`

```
Connected to Cassandra at dse-node1:9042.
[cqlsh 5.0.1 | Cassandra 3.0.7.1159 | DSE 5.0.1 | CQL spec 3.4.0 | Native protocol v4]
Use HELP for help.
cqlsh> CREATE KEYSPACE IF NOT EXISTS ticker WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor' : 2 };
cqlsh>

USE ticker;
Create the table:

CREATE TABLE IF NOT EXISTS symbol_history ( 
  symbol    text,
  year      int,
  month     int,
  day       int,
  volume    bigint,
  close     double,
  open      double,
  low       double,
  high      double,
  idx       text static,
  dummy     text static,
  PRIMARY KEY ((symbol, year), month, day)
) with CLUSTERING ORDER BY (month desc, day desc);
```

Insert Some Records

```
INSERT INTO symbol_history (symbol, year, month, day, volume, close, open, low, high, idx, dummy) 
VALUES ('CORP', 2015, 12, 31, 1054342, 9.33, 9.55, 9.21, 9.57, 'NYSE','CORP_2015') USING TTL 604800;
 
INSERT INTO symbol_history (symbol, year, month, day, volume, close, open, low, high, idx, dummy) 
VALUES ('CORP', 2016, 1, 1, 1055334, 8.2, 9.33, 8.02, 9.35, 'NASDAQ', 'CORP_2016') USING TTL 604800;
 
INSERT INTO symbol_history (symbol, year, month, day, volume, close, open, low, high) 
VALUES ('CORP', 2016, 1, 4, 1054342, 8.54, 8.2, 8.2, 8.65) USING TTL 604800;
 
INSERT INTO symbol_history (symbol, year, month, day, volume, close, open, low, high) 
VALUES ('CORP', 2016, 1, 5, 1054772, 8.73, 8.54, 8.44, 8.75) USING TTL 604800;
```

Update a column value

```
UPDATE symbol_history USING TTL 604800 set close = 8.55 where symbol = 'CORP' and year = 2016 and month = 1 and day = 4;
```

Next, let’s flush memtables to disk as SSTables using nodetool (note the remote JMX port):

`nodetool -p 7299 flush nodetool -p 7299 status`

Then back in a cqlsh session we will set a column value to null and delete an entire row to generate some tombstones:

`cqlsh node2`

Set Column Value To Null

```
USE ticker;
UPDATE symbol_history SET high = null WHERE symbol = 'CORP' and year = 2016 and month = 1 and day = 1;
```

Delete An Entire Row

```
DELETE FROM symbol_history WHERE symbol = 'CORP' and year = 2016 and month = 1 and day = 5;
```

Flush again to generate a new SSTable, and then run a major compaction to create a single SSTable.

```
nodetool -p 7299 flush nodetool -p 7299 compact ticker
```

Now that we have a single SSTable representing operations on our CQL table we can use the appropriate tool to examine its contents.

On-disk representation:

`sstabledump /var/lib/dse-node1/data/ticker/symbol_history-951ebbb154b211e6980f89474b209cfc/mb-3-big-Data.db -d`

```
[CORP:2016]@0 Row[info=[ts=-9223372036854775808] ]: STATIC | [dummy=CORP_2016 ts=1469703689402299 ttl=604800 ldt=1470308489], [idx=NASDAQ ts=1469703689402299 ttl=604800 ldt=1470308489]
[CORP:2016]@0 Row[info=[ts=-9223372036854775808] del=deletedAt=1469730379207681, localDeletion=1469730379 ]: 1, 5 |
[CORP:2016]@90 Row[info=[ts=1469703698486142 ttl=604800, let=1470308498] ]: 1, 4 | [close=8.55 ts=1469703728290212 ttl=604800 ldt=1470308528], [high=8.65 ts=1469703698486142 ttl=604800 ldt=1470308498], [low=8.2 ts=1469703698486142 ttl=604800 ldt=1470308498], [open=8.2 ts=1469703698486142 ttl=604800 ldt=1470308498], [volume=1054342 ts=1469703698486142 ttl=604800 ldt=1470308498]
[CORP:2016]@167 Row[info=[ts=1469703689402299 ttl=604800, let=1470308489] ]: 1, 1 | [close=8.2 ts=1469703689402299 ttl=604800 ldt=1470308489], [high=<tombstone> ts=1469730366699270 ldt=1469730366], [low=8.02 ts=1469703689402299 ttl=604800 ldt=1470308489], [open=9.33 ts=1469703689402299 ttl=604800 ldt=1470308489], [volume=1055334 ts=1469703689402299 ttl=604800 ldt=1470308489]
[CORP:2015]@233 Row[info=[ts=-9223372036854775808] ]: STATIC | [dummy=CORP_2015 ts=1469703676193714 ttl=604800 ldt=1470308476], [idx=NYSE ts=1469703676193714 ttl=604800 ldt=1470308476]
[CORP:2015]@233 Row[info=[ts=1469703676193714 ttl=604800, let=1470308476] ]: 12, 31 | [close=9.33 ts=1469703676193714 ttl=604800 ldt=1470308476], [high=9.57 ts=1469703676193714 ttl=604800 ldt=1470308476], [low=9.21 ts=1469703676193714 ttl=604800 ldt=1470308476], [open=9.55 ts=1469703676193714 ttl=604800 ldt=1470308476], [volume=1054342 ts=1469703676193714 ttl=604800 ldt=1470308476]
```

Or in JSON format:

```
sstabledump /var/lib/dse-node1/data/ticker/symbol_history-951ebbb154b211e6980f89474b209cfc/mb-3-big-Data.db
[
  {
    "partition" : {
      "key" : [ "CORP", "2016" ],
      "position" : 0
    },
    "rows" : [
      {
        "type" : "static_block",
        "position" : 71,
        "cells" : [
          { "name" : "dummy", "value" : "CORP_2016", "tstamp" : "2016-07-28T11:01:29.402299Z", "ttl" : 604800, "expires_at" : "2016-08-04T11:01:29Z", "expired" : false },
          { "name" : "idx", "value" : "NASDAQ", "tstamp" : "2016-07-28T11:01:29.402299Z", "ttl" : 604800, "expires_at" : "2016-08-04T11:01:29Z", "expired" : false }
        ]
      },
      {
        "type" : "row",
        "position" : 71,
        "clustering" : [ "1", "5" ],
        "deletion_info" : { "marked_deleted" : "2016-07-28T18:26:19.207681Z", "local_delete_time" : "2016-07-28T18:26:19Z" },
        "cells" : [ ]
      },
      {
        "type" : "row",
        "position" : 90,
        "clustering" : [ "1", "4" ],
        "liveness_info" : { "tstamp" : "2016-07-28T11:01:38.486142Z", "ttl" : 604800, "expires_at" : "2016-08-04T11:01:38Z", "expired" : false },
        "cells" : [
          { "name" : "close", "value" : "8.55", "tstamp" : "2016-07-28T11:02:08.290212Z" },
          { "name" : "high", "value" : "8.65" },
          { "name" : "low", "value" : "8.2" },
          { "name" : "open", "value" : "8.2" },
          { "name" : "volume", "value" : "1054342" }
        ]
      },
      {
        "type" : "row",
        "position" : 167,
        "clustering" : [ "1", "1" ],
        "liveness_info" : { "tstamp" : "2016-07-28T11:01:29.402299Z", "ttl" : 604800, "expires_at" : "2016-08-04T11:01:29Z", "expired" : false },
        "cells" : [
          { "name" : "close", "value" : "8.2" },
          { "name" : "high", "deletion_info" : { "local_delete_time" : "2016-07-28T18:26:06Z" },
            "tstamp" : "2016-07-28T18:26:06.699270Z"
          },
          { "name" : "low", "value" : "8.02" },
          { "name" : "open", "value" : "9.33" },
          { "name" : "volume", "value" : "1055334" }
        ]
      }
    ]
  },
  {
    "partition" : {
      "key" : [ "CORP", "2015" ],
      "position" : 233
    },
    "rows" : [
      {
        "type" : "static_block",
        "position" : 296,
        "cells" : [
          { "name" : "dummy", "value" : "CORP_2015", "tstamp" : "2016-07-28T11:01:16.193714Z", "ttl" : 604800, "expires_at" : "2016-08-04T11:01:16Z", "expired" : false },
          { "name" : "idx", "value" : "NYSE", "tstamp" : "2016-07-28T11:01:16.193714Z", "ttl" : 604800, "expires_at" : "2016-08-04T11:01:16Z", "expired" : false }
        ]
      },
      {
        "type" : "row",
        "position" : 296,
        "clustering" : [ "12", "31" ],
        "liveness_info" : { "tstamp" : "2016-07-28T11:01:16.193714Z", "ttl" : 604800, "expires_at" : "2016-08-04T11:01:16Z", "expired" : false },
        "cells" : [
          { "name" : "close", "value" : "9.33" },
          { "name" : "high", "value" : "9.57" },
          { "name" : "low", "value" : "9.21" },
          { "name" : "open", "value" : "9.55" },
          { "name" : "volume", "value" : "1054342" }
        ]
      }
    ]
  }
```