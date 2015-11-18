---
layout: post
title:  "Setting up Apache Knox in three easy steps"
date:   2015-11-18
categories: knox
---

This article covers setting up [Apache Knox][knox-site] for development or just to play around with.

# Step 1 - Clone the git repository

```sh
~/Projects> git clone https://git-wip-us.apache.org/repos/asf/knox.git
```

# Step 2 - Build, install and start the servers

```sh
~/Projects> cd knox
~/Projects/knox> ant package install-test-home start-test-servers
```

This will generate a great deal of output.
At the end though you should see something like this.
If not, I've included some debugging tips later below.

```
start-test-ldap:
     [exec] Starting LDAP succeeded with PID 18226.

start-test-gateway:
     [exec] Starting Gateway succeeded with PID 18277.
```

Assuming that the started successfully you can access the Knox Admin API via cURL.

```sh
~/Projects/knox> curl -ku admin:admin-password 'https://localhost:8443/gateway/admin/api/v1/version'
```

This will return an XML response with some version information.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ServerVersion>
   <version>0.7.0-SNAPSHOT</version>
   <hash>fa56190a3de7d33ac07392f81def235bdb2d258c</hash>
</ServerVersion>
```

If the servers failed to start, here are some debugging tips and tricks.

The first thing to check for is other running gateway or ldap servers.
The Java `jps` command is convenient for doing this.
If you find other gateway.jar or ldap.jar processes running this is likely causing the issue.
These will need to be stopped before you can proceed.

```sh
~/Projects/knox> jps
431 Launcher
18277 gateway.jar
18346 Jps
18226 ldap.jar
```

The next likely culprit is some other process running using port required by the gateway (8443) or the demo LDAP server (33389).
On macos the `lsof` command is the tool of choice.
If you find other processes already listening on these ports they will need to be stopped before you can proceed.

```sh
~/Projects/knox> lsof -n -i4TCP:8443 | grep LISTEN
java    18277 kevin.minder  167u  IPv6 0x2d785ee90129816b      0t0  TCP *:pcsync-https (LISTEN)

~/Projects/knox> lsof -n -i4TCP:33389 | grep LISTEN
java    18226 kevin.minder  226u  IPv6 0x2d785ee91fcce56b      0t0  TCP *:33389 (LISTEN)
```

# Step 3 - Customize the topology for your cluster

Once the Knox servers are up and running you need to create or customize topology files match an existing Hadoop cluster.
Please note that your directories may be different than what is shown below depending on what version of Knox you are using.
The version shown here is the 0.7.0-SNAPSHOT version.
Also note that the `open` command is a macos specific command that will likely launch the XML file in Xcode for editing.
Any text editor is fine.

```sh
~/Projects/knox> cd install/knox-0.7.0-SNAPSHOT
~/Projects/knox/install/knox-0.7.0-SNAPSHOT> open conf/topologies/sandbox.xml
```

Right now all you need to worry about are the \<service> sections in the topology file, in particular the \<url> values.
If you are running a local [HDP Sandbox][hdp-sandbox] these values will be correct, otherwise they will need to be changed.

```xml
    <service>
        <role>NAMENODE</role>
        <url>hdfs://localhost:8020</url>
    </service>

    <service>
        <role>JOBTRACKER</role>
        <url>rpc://localhost:8050</url>
    </service>

    <service>
        <role>WEBHDFS</role>
        <url>http://localhost:50070/webhdfs</url>
    </service>

    <service>
        <role>WEBHCAT</role>
        <url>http://localhost:50111/templeton</url>
    </service>

    <service>
        <role>OOZIE</role>
        <url>http://localhost:11000/oozie</url>
    </service>

    <service>
        <role>WEBHBASE</role>
        <url>http://localhost:60080</url>
    </service>

    <service>
        <role>HIVE</role>
        <url>http://localhost:10001/cliservice</url>
    </service>

    <service>
        <role>RESOURCEMANAGER</role>
        <url>http://localhost:8088/ws</url>
    </service>
```

Once you have made the required changes to the \<service> elements save the file.
Within a few seconds the Knox gateway server will detect the change and reload the file.
Then you can access the Hadoop cluster via the gateway with the sample cURL command below.

```sh
curl -ku guest:guest-password 'https://localhost:8443/gateway/sandbox/webhdfs/v1/?op=GETHOMEDIRECTORY' 
```

This should return a response body similar to what is shown below.

```json
{"Path": "/user/guest"}
```

Hopefully this provides the shortest possible path to getting started with Apache Knox.
Most of this information can also be found in the [Apache Knox User's Guide][usr-guide]. 
If you have more questions, comments or suggestions please join the [Apache Knox][knox-site] community.
In particular you might be interested in one of the [mailing lists][knox-lists].

[knox-site]: http://knox.apache.org/
[knox-lists]: http://knox.apache.org/mail-lists.html
[usr-guide]: http://knox.apache.org/books/knox-0-6-0/user-guide.html "Apache Knox User's Guide"
[dev-guide]: http://knox.apache.org/books/knox-0-6-0/dev-guide.html "Apache Knox Developer's Guide"
[hdp-sandbox]: http://hortonworks.com/products/hortonworks-sandbox/
