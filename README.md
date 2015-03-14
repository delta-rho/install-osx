# Tessera on OS X

While the purpose of the [Vagrant VMs](https://github.com/tesseradata/install-vagrant) is mainly to provide an isolated, reproducible development VM, as well as a sandbox to play around in, it can be nice to have a development environment set up directly on your machine.  This README goes through some steps for setting up components of Tessera on OS X.

The goal here is to provide a mostly-self-contained installation of several of the Tessera components without needing to scatter files all over the computer or create new users for Hadoop, Spark, etc.  The resulting installation is not what you'd expect to see on any enterprise system, but it gets the job done for development.

Note that since the steps here will modify your own system, this is only recommended if you comfortable with the command line.  Follow the steps carefully to ensure they are appropriate for your system, as you are responsible for the changes you make to your system.

### Prerequisites ###

It is assumed you already have the following installed:

- R 3.x
- XCode command line tools
- Homebrew
- git

Also, the steps below have only been validated on a machine running OS X Yosemite, and all commands are done in bash shell.

### Components ###

The major components to be installed are:

- Hadoop 2.4
- Spark 1.2
- Protocol buffers 2.5
- RHIPE 0.75
- SparkR

### Step 1. Preliminaries ###

#### Passwordless SSH

Everything will be installed and run by your user (no root).  You need to be able to do passwordless ssh into your own machine, which you can do with the following:

```
ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa
cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
```

#### Dependencies

We need protocol buffers 2.5, pkg-config, maven, and ant, which we'll install with `brew`:

```
brew install homebrew/versions/protobuf250
brew install pkg-config
brew install maven
brew install ant
```

### Step 2. Hadoop / RHIPE ###

All the files for Hadoop and Spark will go in a single directory.  Here, we call this directory `hadoop` and put it in our home directory:

```
mkdir $HOME/hadoop
cd $HOME/hadoop
wget http://archive.apache.org/dist/hadoop/core/hadoop-2.4.0/hadoop-2.4.0.tar.gz
tar -xzf hadoop-2.4.0.tar.gz
cd hadoop-2.4.0

export HADOOP_HOME=$HOME/hadoop/hadoop-2.4.0
```

#### Edit configuration files

Open `$HADOOP_HOME/etc/hadoop/core-site.xml` with your favorite editor and put the following inside the `<configuration>` block:

```xml
<property>
  <name>fs.defaultFS</name>
  <value>hdfs://localhost:8020</value>
</property>
<property>
    <name>hadoop.tmp.dir</name>
    <value>/Users/__username__/hadoop/hadooptmpdir</value>
</property>
<!-- Mapred proxy user setting -->
<property>
  <name>hadoop.proxyuser.mapred.hosts</name>
  <value>*</value>
</property>
<property>
  <name>hadoop.proxyuser.mapred.groups</name>
  <value>*</value>
</property>
```

Note that you need to replace `__username__` with your username for `hadoop.tmp.dir`.  Or you can specify whatever location you would like.

Now rename the `mapred-site.xml.template`

```bash
mv $HADOOP_HOME/etc/hadoop/mapred-site.xml.template $HADOOP_HOME/etc/hadoop/mapred-site.xml
```

and set the following in it:

```xml
<property>
  <name>mapred.job.tracker</name>
  <value>localhost:8021</value>
</property>
<property>
  <name>mapreduce.framework.name</name>
  <value>yarn</value>
</property>
<property>
  <name>mapreduce.jobhistory.address</name>
  <value>localhost:10020</value>
</property>
<property>
  <name>mapreduce.jobhistory.webapp.address</name>
  <value>localhost:19888</value>
</property>
```

In `$HADOOP_HOME/etc/hadoop/hadoop-env.sh`, make sure the following are set:

```bash
export JAVA_HOME="$(/usr/libexec/java_home -v 1.6)"
# Extra Java runtime options.  Empty by default.
export HADOOP_OPTS="$HADOOP_OPTS -Djava.net.preferIPv4Stack=true"
export HADOOP_OPTS="$HADOOP_OPTS -Djava.awt.headless=true -Djava.security.krb5.realm=-Djava.security.krb5.kdc="
export YARN_OPTS="$YARN_OPTS -Djava.security.krb5.realm=OX.AC.UK -Djava.security.krb5.kdc=kdc0.ox.ac.uk:kdc1.ox.ac.uk -Djava.awt.headless=true"
```

Edit `$HADOOP_HOME/etc/hadoop/yarn-site.xml` to have the following property:

```xml
<property>
  <name>yarn.nodemanager.aux-services</name>
  <value>mapreduce_shuffle</value>
</property>
```

#### Fire things up ####

```bash
# environment variables
export HADOOP_HOME=$HOME/hadoop/hadoop-2.4.0
export PATH=$PATH:$HADOOP_HOME/bin
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop

# stop things in case they have been started from a previous pass
$HADOOP_HOME/sbin/mr-jobhistory-daemon.sh stop historyserver
$HADOOP_HOME/sbin/stop-yarn.sh
$HADOOP_HOME/sbin/stop-dfs.sh

# remove hadooptmpdir and format namenode
rm -rf $HOME/hadoop/hadooptmpdir
hdfs namenode -format

# start namenode, secondary namenode and datanode
$HADOOP_HOME/sbin/start-dfs.sh
# make sure they are running
jps
# also go to http://localhost:50070/ to verify it started

# make some directories on HDFS
hadoop fs -mkdir -p /tmp/hadoop-yarn/staging/history/done_intermediate
hadoop fs -chmod -R 1777 /tmp
hadoop fs -mkdir -p /var/log/hadoop-yarn
hadoop fs -mkdir -p /user/hafen

# start YARN and historyserver
$HADOOP_HOME/sbin/start-yarn.sh
$HADOOP_HOME/sbin/mr-jobhistory-daemon.sh start historyserver --config $HADOOP_CONF_DIR
# check that they are running
jps

# also check here: http://localhost:8088/cluster/nodes

# test a hadoop job to make sure everything works
hadoop fs -mkdir input
hadoop fs -put $HADOOP_HOME/etc/hadoop/*.xml input
hadoop fs -ls input

hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.4.0.jar grep input output 'dfs[a-z.]+'

hadoop fs -cat output/part-r-00000
```

#### Install RHIPE ####

Make sure you have installed the prerequisites listed previously (protobuf, etc.).

You can either grab it and install:

```bash
wget http://ml.stat.purdue.edu/rhipebin/Rhipe_0.75.1_hadoop-2.tar.gz
R CMD INSTALL Rhipe_0.75.1_hadoop-2.tar.gz
```

Or clone and build it (need ant and maven installed):

```bash
git clone https://github.com/tesseradata/RHIPE
cd RHIPE
ant clean
ant build-distro -Dhadoop.version=hadoop-2
R CMD INSTALL Rhipe_0.75.1_hadoop-2.tar.gz
```

Note that the version number in `R CMD INSTALL` may be different in this case as this will be the latest version.

Now you can check that it installed and run a simple test with with the following:

```s
R
# to be run inside R console:
library(Rhipe)
rhinit()
library(testthat)
test_package("Rhipe", "simple")
```

### Step 3. Spark / SparkR ###

If you'd like to add Spark / SparkR to your installation, you can follow these steps.

#### Install Spark ####

```bash
cd $HOME/hadoop
wget http://www.apache.org/dyn/closer.cgi/spark/spark-1.2.0/spark-1.2.0-bin-hadoop2.4.tgz
tar -xzf spark-1.2.0-bin-hadoop2.4.tgz
cd spark-1.2.0-bin-hadoop2.4
mv conf/spark-env.sh.template conf/spark-env.sh
chmod 755 conf/spark-env.sh
```

Now set the following in `conf/spark-env.sh`:

```bash
SPARK_MASTER_OPTS="$SPARK_MASTER_OPTS -Djava.awt.headless=true"
SPARK_WORKER_OPTS="$SPARK_WORKER_OPTS -Djava.awt.headless=true"
```

Now start master and worker.  Note that you will need to set `__machine_name__` accordingly (see "Computer name:" in the "Sharing" preference pane in "System Preferences").

```bash
./sbin/start-master.sh
./bin/spark-class org.apache.spark.deploy.worker.Worker spark://__machine_name__.local:7077 &
```

#### Install SparkR ####

```bash
git clone https://github.com/amplab-extras/SparkR-pkg
cd SparkR-pkg
# SPARK_VERSION=1.2.0 SPARK_YARN=1 SPARK_HADOOP_VERSION=2.4.0 SPARK_YARN_VERSION=2.4.0 ./install-dev.sh
SPARK_VERSION=1.2.0 SPARK_HADOOP_VERSION=2.4.0 ./install-dev.sh
rm -rf ~/Library/R/3.1/library/SparkR
cp -R lib/SparkR ~/Library/R/3.1/library/
```

This assumes your R package library is in `~/Library/R/3.1/library`.  Adjust according to your environment.

Make sure it worked:

```bash
R
library(SparkR)
sc <- sparkR.init(master="spark://__machine_name__.local:7077",
  sparkEnvir=list(spark.executor.memory="1g"))

rdd <- parallelize(sc, 1:10, 2)
length(rdd)
```

### Step 4. Managing the Components ###

#### Environment Variables ####

Here is a list of environment variables we have set throughout.  If you open up a new shell session you can paste all of these in.  Or you can set these in your `.bash_profile` if you'd like.

```bash
## environment variables
export HADOOP_HOME=$HOME/hadoop/hadoop-2.4.0
export PATH=$PATH:$HADOOP_HOME/bin
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export HADOOP_BIN=$HADOOP_HOME/bin
export HADOOP_LIBS=`hadoop classpath | tr -d '*'`
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export HADOOP_CMD=$HADOOP_BIN/hadoop
export SPARK_HOME=$HOME/hadoop/spark-1.2.0-bin-hadoop2.4
```

#### Stopping and Starting Everything ####

For convenience, here are the commands to stop and start everything:

```bash
## stop things:
$HADOOP_HOME/sbin/mr-jobhistory-daemon.sh stop historyserver
$HADOOP_HOME/sbin/stop-yarn.sh
$HADOOP_HOME/sbin/stop-dfs.sh
$SPARK_HOME/sbin/stop-slaves.sh
$SPARK_HOME/sbin/stop-master.sh

## start things:
$HADOOP_HOME/sbin/start-dfs.sh
$HADOOP_HOME/sbin/start-yarn.sh
$HADOOP_HOME/sbin/mr-jobhistory-daemon.sh start historyserver --config $HADOOP_CONF_DIR
$SPARK_HOME/sbin/start-master.sh
$SPARK_HOME/sbin/start-slave.sh 1 spark://__machine_name__.local:7077
```


