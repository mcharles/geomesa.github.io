---
title: Deploying GeoMesa on HDP Sandbox 2.2.4
author: mcharles
layout: tutorial
---

{% include tutorial-header.html %}

##### This tutorial will introduce how to deploy GeoMesa on a Hortonworks Data Platform (HDP) virtual machine in six stages:

1. Mount HDP with VirtualBox
2. Install Zookeeper & Accumulo
3. Build GeoMesa
4. Install Apache Tomcat 7
5. Install GeoServer
6. Deploy GeoMesa to GeoServerDeploy GeoMesa on a Hortonworks Data Platform (HDP) virtual machine

<!--more-->



<br>
### Stage 1: Mount HDP on Virtual Machine

##### 1.1
Download and install [VirtualBox](https://www.virtualbox.org/wiki/Downloads)

##### 1.2
Download [HDP 2.2.4](http://hortonworks.com/products/hortonworks-sandbox/#install) image for VirtualBox

Right-click the downloaded image and select "Open with VirtualBox"

##### 1.3
Right-click the image in VirtualBox and selet 'Start'

<br>
##### Stage 1 Checkpoint
  - Enter '127.0.0.1:8888' in browser and to be directed to the Hortonworks Sandbox registration page
  - Enter 'localhost:50070/dfshealth.jsp' to view Hadoop NameNode status


<br>
### Stage 2: Install Zookeeper & Accumulo

##### 2.1
In a terminal window, enter the following to access the HDP VM:

```
ssh root@127.0.0.1 -p 2222
```
password: hadoop

##### 2.2

Install Zookeeper and Accumulo by entering:

```
yum install zookeeper
yum install accumulo
```

##### 2.3

Set directories for Zookeeper, Accumulo, and Hadoop:

```
export ZOOKEEPER_HOME=/usr/hdp/2.2.4.2-2/zookeeper
export ACCUMULO_HOME=/usr/hdp/2.2.4.2-2/accumulo
export HADOOP_PREFIX=/usr/hdp/2.2.4.2-2/hadoop
export HADOOP_HOME=/usr/hdp/2.2.4.2-2/hadoop
```

##### 2.4
Switch to the HDFS user to make an Accumulo data directory and set appropriate permissions
```
su hdfs
hadoop fs -mkdir -p /apps/accumulo
hadoop fs -chmod -R 700 /apps/accumulo
hadoop fs -chown -R accumulo:accumulo /apps/accumulo
```
Switch back to the root user ('su root')


##### 2.5
Copy the example configuration files into the conf directory:
```
cp $ACCUMULO_HOME/conf/examples/512MB/standalone/* $ACCUMULO_HOME/conf
```

##### 2.6
Edit and save ```$ACCUMULO_HOME/conf/accumulo-site.xml``` to include the following:
```
<property>
    <name>instance.volumes</name>
    <value>hdfs://sandbox:8020/apps/accumulo</value>
</property>`
```
##### 2.7
Edit the following 5 files, located in ```$ACCUMULO_HOME/conf```, and replace "localhost" with "sandbox":

- gc
- masters
- monitor
- tracers
- slaves

##### 2.8

Initialize Accumulo:
```
$ACCUMULO_HOME/bin/accumulo init
```
Enter an instance name (e.g. accumulo) and password (e.g. secret)

Start Accumulo:
```
$ACCUMULO_HOME/bin/start-all.sh
```
<br>
##### Stage 2 Checkpoint
  - Enter '127.0.0.1:50095' in browser and to be directed to an Accumulo overview w/ Zookeeper at port 2181

<br>
### Stage 3: Build GeoMesa & Test Accumulo QuickStart


##### Syntax for transferring files from local machine to HDP Sandbox via SCP:
```
scp -P 2222 local/file/path/filename root@127.0.0.1:vm/desired/filepath
```

##### 3.1

On your local machine, clone the [GeoMesa github branch](https://github.com/locationtech/geomesa/tree/jnh_accumulo1.6) for Accumulo 1.6, and build using ```mvn clean install```

##### 3.2

Find ```geomesa-1.1.0-rc.3-SNAPSHOT.jar``` in the GeoMesa build on local machine (located in geomesa-assemble/target) and SCP to the VM
```
scp -P 2222 geomesa-assemble/target/geomesa-1.1.0-rc.3-SNAPSHOT-bin.tar.gz root@127.0.0.1:/usr/hdp/2.2.4.2-2/
```
On the VM, untar the file to create a GeoMesa directory (e.g. /usr/hdp/2.2.4.2-2/geomesa-1.1.0-rc.3-SNAPSHOT) and set GEOMSA_HOME:

```
export GEOMESA_HOME=/usr/hdp/2.2.4.2-2/geomesa-1.1.0-rc.3-SNAPSHOT
export PATH=${GEOMESA_HOME}/bin:$PATH
```

##### 3.3

In the GeoMesa directory, run configure:
```
bin/geomesa configure
```

Install GPL software:
```
bin/install-jai
bin/install-jline
bin/install-vecmath
```

##### 3.4

Deploy GeoMesa to Accumulo by copying the distributed runtime jar into Accumulo's lib/ext directory
```
scp $GEOMESA_HOME/dist/geomesa-distributed-runtime-1.1.0-rc.3-SNAPSHOT.jar $ACCUMULO_HOME/lib/ext
```
##### 3.5
CD into GeoMesa build on local machine and SCP ```geomesa-accumulo-quickstart-1.1.0-rc.3-SNAPSHOT.jar``` (located in geomesa-examples/geomesa-accumulo-quickstart/target) to VM
```
scp -P 2222 geomesa-examples/geomesa-accumulo-quickstart/target/geomesa-accumulo-quickstart-1.1.0-rc.3-SNAPSHOT.jar root@127.0.0.1:/usr/hdp/2.2.4.2-2/
```
##### 3.6
Run AccumuloQuickStart with the following command:

{% highlight bash %}
java -cp ./geomesa-accumulo-quickstart-1.1.0-rc.3-SNAPSHOT.jar org.locationtech.geomesa.examples.AccumuloQuickStart -instanceId accumulo -zookeepers "localhost:2181" -user root -password secret -tableName VMtest
{% endhighlight %}

See [GeoMesa Quick Start](http://www.geomesa.org/geomesa-quickstart/) for more details on this step
<br>
##### Stage 3 Checkpoint
  - The above command should yield the output similar to:

{% highlight bash %}
'''
Creating feature-type (schema):  AccumuloQuickStart
Creating new features
Inserting new features
Submitting query
1.  Bierce|640|Sun Sep 14 15:48:25 EDT 2014|POINT (-77.36222958792739 -37.13013846773835)|null
2.  Bierce|886|Tue Jul 22 14:12:36 EDT 2014|POINT (-76.59795732474399 -37.18420917493149)|null
3.  Bierce|925|Sun Aug 17 23:28:33 EDT 2014|POINT (-76.5621106573523 -37.34321201566148)|null
4.  Bierce|589|Sat Jul 05 02:02:15 EDT 2014|POINT (-76.88146600670152 -37.40156607152168)|null
5.  Bierce|394|Fri Aug 01 19:55:05 EDT 2014|POINT (-77.42555615743139 -37.26710898726304)|null
6.  Bierce|931|Fri Jul 04 18:25:38 EDT 2014|POINT (-76.51304097832912 -37.49406125975311)|null
7.  Bierce|322|Tue Jul 15 17:09:42 EDT 2014|POINT (-77.01760098223343 -37.30933767159561)|null
8.  Bierce|343|Wed Aug 06 04:59:22 EDT 2014|POINT (-76.66826220670282 -37.44503877750368)|null
9.  Bierce|259|Thu Aug 28 15:59:30 EDT 2014|POINT (-76.90122194030118 -37.148525741002466)|null
'''
{% endhighlight %}

<br>
### Stage 4: Install Apache Tomcat 7

##### 4.1

Download the Core tar.gz for [Apache Tomcat 7](https://tomcat.apache.org/download-70.cgi)

SCP the tar.gz file to VM and tar to create Tomcat directory

```
scp -P 2222 apache-tomcat-7.0.62.tar.gz root@127.0.0.1:/usr/hdp/2.2.4.2-2/
tar xzf apache-tomcat-7.0.62.tar.gz
```

##### 4.2
CD into new Tomcat folder and start Tomcat with:
```
bin/startup.sh
```

<br>
##### Stage 4 Checkpoint
  - Navigate to 127.0.0.1:8080  in browser to confirm Apache Tomcat is running

<br>
### Stage 5: Install GeoServer

##### 5.1

Download [Geoserver-2.5.2.war](http://sourceforge.net/projects/geoserver/files/GeoServer/2.5.2/geoserver-2.5.2-war.zip/download) and unzip

##### 5.2

SCP ```geoserver.war``` into Tomcat's webapps directory:
```
scp -P 2222 geoserver.war root@127.0.0.1:/usr/hdp/2.2.4.2-2/apache-tomcat-7.0.62/webapps
```

Shutdown and restart Tomcat
<br>
##### Stage 5 Checkpoint
  - Navigate to 127.0.0.1:8080/geoserver and login with username: admin / password: geoserver



<br>
### Stage 6: Deploy GeoMesa to GeoServer

##### 6.1
Copy GeoMesa plugin to GeoServer:
```
cp $GEOMESA_HOME/dist/geomesa-plugin-1.1.0-rc.3-SNAPSHOT-geoserver-plugin.jar /usr/hdp/2.2.4.2-2/apache-tomcat-7.0.62/webapps/geoserver/WEB-INF/lib
```

##### 6.2

Install additional dependencies as described in [GeoMesa Deployment](http://www.geomesa.org/geomesa-deployment/)

The HDP sandbox VM is using Accumulo 1.6, Zookeeper 3.4.6, and Hadoop 2.6, so use the following download links to ensure correct versions:

###### Accumulo
accumulo-core-1.6.0.jar[Download](http://search.maven.org/remotecontent?filepath=org/apache/accumulo/accumulo-core/1.6.0/accumulo-core-1.6.0.jar)

accumulo-fate-1.6.0.jar [Download](http://search.maven.org/remotecontent?filepath=org/apache/accumulo/accumulo-fate/1.6.0/accumulo-fate-1.6.0.jar)

accumulo-trace-1.6.0.jar [Download](http://search.maven.org/remotecontent?filepath=org/apache/accumulo/accumulo-trace/1.6.0/accumulo-trace-1.6.0.jar)

###### Zookeeper
zookeeper-3.4.6.jar [Download](https://repo1.maven.org/maven2/org/apache/zookeeper/zookeeper/3.4.6/zookeeper-3.4.6.jar)

###### Hadoop core
hadoop-auth-2.6.0.jar [Download](http://central.maven.org/maven2/org/apache/hadoop/hadoop-auth/2.6.0/hadoop-auth-2.6.0.jar)

hadoop-client-2.6.0.jar [Download](http://central.maven.org/maven2/org/apache/hadoop/hadoop-client/2.6.0/hadoop-client-2.6.0.jar)

hadoop-common-2.6.0.jar [Download](http://central.maven.org/maven2/org/apache/hadoop/hadoop-common/2.6.0/hadoop-common-2.6.0.jar)

hadoop-hdfs-2.6.0.jar [Download](http://central.maven.org/maven2/org/apache/hadoop/hadoop-hdfs/2.6.0/hadoop-hdfs-2.6.0.jar)

hadoop-mapreduce-client-app-2.6.0.jar [Download](http://central.maven.org/maven2/org/apache/hadoop/hadoop-mapreduce-client-app/2.6.0/hadoop-mapreduce-client-app-2.6.0.jar)

hadoop-mapreduce-client-common-2.6.0.jar [Download](http://central.maven.org/maven2/org/apache/hadoop/hadoop-mapreduce-client-common/2.6.0/hadoop-mapreduce-client-common-2.6.0.jar)

hadoop-mapreduce-client-core-2.6.0.jar [Download](http://central.maven.org/maven2/org/apache/hadoop/hadoop-mapreduce-client-core/2.6.0/hadoop-mapreduce-client-core-2.6.0.jar)

hadoop-mapreduce-client-jobclient-2.6.0.jar [Download](http://central.maven.org/maven2/org/apache/hadoop/hadoop-mapreduce-client-jobclient/2.6.0/hadoop-mapreduce-client-jobclient-2.6.0.jar)

hadoop-mapreduce-client-shuffle-2.6.0.jar [Download](http://central.maven.org/maven2/org/apache/hadoop/hadoop-mapreduce-client-shuffle/2.6.0/hadoop-mapreduce-client-shuffle-2.6.0.jar)

###### Thrift
libthrift-0.9.1.jar [Download](https://search.maven.org/remotecontent?filepath=org/apache/thrift/libthrift/0.9.1/libthrift-0.9.1.jar)

SCP all of these files into GeoServer's ```WEB-INF/lib``` directory


##### 6.3

Two pre-existing JARs must also be updated in the lib directory:

commons-configuration: Accumulo requires commons-configuration 1.6 and previous versions should be replaced [Download](https://search.maven.org/remotecontent?filepath=commons-configuration/commons-configuration/1.6/commons-configuration-1.6.jar)

commons-lang: GeoServer ships with commons-lang 2.1, but Accumulo requires replacing that with version 2.4 [Download](https://search.maven.org/remotecontent?filepath=commons-lang/commons-lang/2.4/commons-lang-2.4.jar)


<br>
##### Tutorial Completion Check
  - Execute the steps under 'Visualize Data With Geoserver' in the [GeoMesa Quick Start](http://www.geomesa.org/geomesa-quickstart/)
