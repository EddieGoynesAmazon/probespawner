
# What’s probespawner
Probespawner is a small jython program initially designed to repeat JDBC queries periodically and write it’s output to an Elasticsearch cluster, file or STDOUT, but any can be added.
Examples of parsers for “top”, “netstat -s”, “netstat -ntce”, command execution and what not have been packaged.
It’s immature, and since other challenges are being pursued, it’s published for everyone to mature or serve as an example for whatever.
It get's usefull sometimes when troubleshooting.

# Why probespawner
Policies forbidding running rivers and installing plugins in the elasticsearch nodes were instated.
Probespawner got written initally to perform some tasks that [elasticsearch-river-jdbc](https://github.com/jprante/elasticsearch-river-jdbc) feeder did not address and to come around the bugs and difficulties if setting up one such feeder.
Other work extended from there to support monitoring and performance statistics on the OSes.

# How does probespawner work
Probespawner reads a JSON configuration file stating a list of inputs and the outputs for each, much like [logstash](https://www.elastic.co/products/logstash).
The inputs provided are either JMX (probing a JVM), JDBC (querying a database) or execution of programs in different platforms.
Each is called a probe.
The data acquired cyclically from these input sources are sent to Elasticsearch, stdout or file.

Basically, for each input you have defined, probespawner will launch a (java) thread as illustrated in the concurrency manual of jython.
Each thread is an instance of a probe that performs:
* Periodical acquisition of query results from a database and writes these to an Elasticsearch instance (using Elasticsearch’s JAVA api).
* Periodical acquisition of JMX attributes from a JVM instance writing these to an index of your choice on your Elasticsearch cluster and to a file you designated.
* Periodically executes any task you designed for your own probe and do whatever you want with the results, for instance, write them to STDOUT.

# Installing

## Dependencies

1. Jython 2.5.3+ - http://www.jython.org/downloads.html
2. Jyson 1.0.2+ - http://opensource.xhaus.com/projects/jyson
3. JodaTime 2.7+ - https://github.com/JodaOrg/joda-time

### Optional but real useful
1. Tomcat’s 7.0.9+ (connection pool classes) - http://tomcat.apache.org/download-70.cgi
2. Elasticsearch 1.5.0+ - https://www.elastic.co/downloads

### JDBC drivers you need for your queries, some common ones for your reference:
1. Mysql - http://dev.mysql.com/downloads/connector/j/
2. Oracle - http://www.oracle.com/technetwork/apps-tech/jdbc-112010-090769.html
3. MSSQL - http://www.microsoft.com/en-us/download/details.aspx?displaylang=en&id=11774, http://go.microsoft.com/fwlink/?LinkId=245496

About the use of Tomcat’s connection pool, zxJDBC could’ve been used to attain the same objective.
Since some code was around using it, so it stood.

## Installing probespawner
Just grab it from git hub or expand the tarball/zip you’ve downloaded.
Inside you have the probespawner launchers (.sh or .bat) to start the program depending on your OS (Linux/Mac/Windows).

The tarball contains the following files:

1. **dummyprobe.py** - Do not be mistaken judging by the name that this is a dummy probe. It’s been a poor choice of name but this is immature code so it’s staying like that. This is the class from which the other inherit and it’s the real skeleton for probes.
2. **cooldbprobe.py** - Same as databaseprobe.py but uses JodaTime for time parameters, timestamp is in milliseconds, datetime strings are ISO8601 format looking like “2015-11-21T15:14:15.912+05:00”.
3. **jelh.py** - Jython’s Elasticsearch Little Helper, the module responsible for the bulk requests to elasticsearch. 
4. **jmxprobe.py** - This probe executes JMX requests in a JVM and reports it’s results to the outputs configured in the JSON setup. NOTE: It discards bad objects/attributes in case of failure but keeps working if it happens. Adjust to your needs (find out the try/except in the “tick” method)
5. **probespawner.ini** - Logging configuration.
6. **probespawner.py** - It’s the croupier, it reads and delivers the configuration for the probes. Lastly, it instantiates all of them and wait’s for them to finish. It’s a simple threadpool but it would be great if it dealt with interrupts and failures from it’s workers.
7. **probespawner.sh** - Launches probespawner with it’s argument as configuration.
8. **probespawner.bat** - Launches probespawner with it’s argument as configuration in case you’re using Microsoft Windows.
9. **testprobe.py** - The real dummy probe, it merely states that is a test and sends a sample dictionary to the outputs configured.
10. **example.json** - Sample JSON configuration for a JDBC and JMX input
11. **zxdatabaseprobe.py** - Same as databaseprobe.py but dispensing the use of Tomcat’s connection pool (uses zxJDBC)
12. **databaseprobe.py** - Amazingly, this does not depend from “dummyprobe.py”, it was the initial code of a more monolitic probe that existed in the past. This probe executes any given query in a database and reports it’s results (if any) to the outputs configured in the JSON setup. It's here for legacy reasons, you should ignore this probe.
13. **execprobe.py** - Executes “command” every cycle and reports it’s output
14. **linuxtopprobe.py** - Executes top command on linux boxes every cycle, parses and reports it’s output in an elasticsearch friendly fashion
15. **netstats.py** - Executes “netstat -s” command on linux boxes every cycle, parses and reports it’s output in an elasticsearch friendly fashion.
16. **netstatntc.py** - Executes “netstat -ntce” command on linux boxes every cycle, parses and reports it’s output in an elasticsearch friendly fashion.

# Configuring
At the end you’ll find a working JSON file for your reference.
Where omitted a description of a field it means it has no action.
The list of possible fields for inputs and outputs is shown below:


-- to be available, refer to the example.json file in the meantime --



# Running probespawner

## Export the classpath:
`export CLASSPATH=/home/suchuser/opt/apache-tomcat-7.0.59/lib/tomcat-jdbc.jar:/home/suchuser/opt/apache-tomcat-7.0.59/bin/tomcat-juli.jar:/home/suchuser/var/lib/java/jyson-1.0.2/lib/jyson-1.0.2.jar:/home/suchuser/opt/elasticsearch-1.5.0/lib/lucene-core-4.10.4.jar:/home/suchuser/opt/elasticsearch-1.5.0/lib/elasticsearch-1.5.0.jar:/home/suchuser/var/lib/java/mysql-connector-java-5.1.20-bin.jar:/home/suchuser/var/lib/java/sqljdbc_4.0/enu/sqljdbc4.jar:/home/suchuser/var/lib/java/sqljdbc_4.0/enu/sqljdbc.jar`

## Run the jython code:
`jython probespawner.py --configuration=example.json`

## Shell script launcher
This one calls “find” in the CWD and adds all *jar files to the classpath before calling jython.  
`probespawner.sh example.json`

## Batch file launcher
The batch file lists all jars in the “jars” directory and adds them to the classpath before calling jython.  
`probespawner.bat example.json`

# Developing other probes
Just in case you want to develop a probe for your convenience, be warned: this is immature code with no support.
Having said that, take a look at the “testprobe.py” file.

## Produce your data: override the “tick” method
This is the minimum probe, the only thing it does is writing the `{ ‘test’: ‘yes’ }` dictionary to the configured outputs in your JSON file.
For that to be accomplished, your class must override the “tick” method which is called at every cycle, meaning every “interval” seconds you defined in your input.

## Get your configuration and initialize your probe: override the “initialize” method
The method “initialize” is another popular overridden one, called before the probe starts cycling. 
It’s usually in this method you get the configuration keys from the JSON file, set up and initialize your probe.

## Getting a property:
`username = self.getInputProperty("username")`  
This grabs the “username” content of your defined input in the JSON file and returns it’s value (or None).


