---
layout: post
title:  "Using Apache Knox with ActiveDirectory"
date:   9999-01-01
categories: knox
---

This article covers using [Apache Knox][knox-site] with ActiveDirectory.

Currently Apache Knox comes "out of the box" setup with a demo LDAP server based on [ApacheDS][apache-ds].
This was a conscious decision made to simplify the initial user experience with Knox.
Unfortunately, it can make the transition to popular enterprise identity stores such as ActiveDirectory confusing.
This article is intended to remedy some of that confusion.

## Part 1
 
So lets go back to basics and build up an example from first principles.
To do this we will start with the simplest topology file that will work.
We will iteratively transform that topology file until it integrates with ActiveDirectory for both authentication and authorization.
 
# Sample 1

The initial topology file we will start with doesn't integrate with ActiveDirectory at all.
Create this topology file.
 
[\<GATEWAY_HOME>/conf/topologies/sample1.xml](/static/activedirectory/sample1.xml)

```xml
<topology>
  <gateway>

    <provider>
      <role>authentication</role>
      <name>ShiroProvider</name>
      <enabled>true</enabled>
      <param name="users.admin" value="admin-secret"/>
      <param name="urls./**" value="authcBasic"/>
    </provider>

  </gateway>
  <service>
    <role>KNOX</role>
  </service>
</topology>
```

If you are a seasoned Knox veteran, you may notice the alternative \<param name="" value=""/> style syntax.
Both this and \<param>\<name>\</name>\<value>\</value>\</param> style are supported.
I've used the attribute style here for compactness.  

Once this topology file is created you will be able to access the Knox Admin API which is what the KNOX service in the topology file provides.
The cURL command shown below retrieves the version information from the Knox server.
Notice `-u admin:admin-secret` in the command below matches `<param name="users.admin" value="admin-secret"/>` in the topology file above.  

```sh
curl -u admin:admin-secret -ik 'https://localhost:8443/gateway/sample1/api/v1/version'
```

Below is an example response body output from the command above.  
_Note: The `-i` causes the return of the full response including status line and headers which aren't shown below._

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ServerVersion>
   <version>0.7.0-SNAPSHOT</version>
   <hash>9632b697060bfeffa2e03425451a3e9b3980c45e</hash>
</ServerVersion>
```

As an aside, if you prefer JSON you can request that using the HTTP Accept header via the cURL `-H` flag.  
Don't forget to scroll right as some of these commands will start to get long.

```sh
curl -u admin:admin-secret -H 'Accept: application/json' -ik 'https://localhost:8443/gateway/sample/api/v1/version'
```

Below is an example response JSON body.

```json
{
   "ServerVersion" : {
      "version" : "0.7.0-SNAPSHOT",
      "hash" : "9632b697060bfeffa2e03425451a3e9b3980c45e"
   }
}
```

# Sample 2

With authentication working lets now add authorization since the ultimate goal is an example with ActiveDirectory including both.
The second sample topology file below adds a second user and an authorization provider.
The `<param name="knox.acl" value="admin;*;*"/>` dictates that only the admin user can access the knox service.
Create this topology file.

[\<GATEWAY_HOME>/conf/topologies/sample2.xml](/static/activedirectory/sample2.xml)

```xml
<topology>
  <gateway>

    <provider>
      <role>authentication</role>
      <name>ShiroProvider</name>
      <enabled>true</enabled>
      <param name="users.admin" value="admin-secret"/>
      <param name="users.guest" value="guest-secret"/>
      <param name="urls./**" value="authcBasic"/>
    </provider>

    <provider>
      <role>authorization</role>
      <name>AclsAuthz</name>
      <enabled>true</enabled>
      <param name="knox.acl" value="admin;*;*"/>
    </provider>

  </gateway>
  <service>
    <role>KNOX</role>
  </service>
</topology>
```

Once this is created, test it with the cURL commands below and see that the admin user can access the API but the guest user can't.

```sh
curl -u admin:admin-secret -ik 'https://localhost:8443/gateway/sample2/api/v1/version'
```

```sh
curl -u guest:guest-secret -ik 'https://localhost:8443/gateway/sample2/api/v1/version'
```

The second command above will return a `HTTP/1.1 403 Forbidden` status along with an error response body.

## Part 2

These embedded examples are all well and good but this article is supposed to be about ActiveDirectory.
This however takes us from examples that will "just work" to examples that need to be customized for the environment in which they run.
Specifically they require some basic network address and LDAP information.
The table below describes the information you will need from your environment and shows what is being used in the samples here.
You will need to adjust these in the samples to match your environment.

**A word of caution is warranted here.
There are as many ways to setup LDAP and ActiveDirectory as there are IT departments.
This variability requires flexible configuration which in turn often causes confusion, especially given poor documentation. 
The examples here focus on a single specific pattern that is seen frequently, but your mileage may vary.**

| Name             | Description                                                         | Example                                                |
| ---------------- | ------------------------------------------------------------------- | ------------------------------------------------------ |
| Server Host      | The hostname where ActiveDirectory is running.                      | ad.qa.your-domain.com                                  |
| Server Port      | The port on which ActiveDirectory is listening.                     | 389                                                    |
| System Username  | The distinguished name for a user with search permissions.          | CN=Kevin Minder,CN=Users,DC=hwqe,DC=hortonworks,DC=com |
| System Password  | The password for the system user. (See: Note1)                      | ********                                               |
| Search Base      | The subset of users to search for authentication. (See: Note2)      | CN=Users,DC=hwqe,DC=hortonworks,DC=com                 |
| Search Attribute | The attribute containing the username to search for authentication. | sAMAccountName                                         |
| Search Class     | The object class for LDAP entities to search for authentication.    | person                                                 |
<br>
Note1: In these samples the password will be embedded with the topology files.
This is for simplicity.
The password can be stored in a protected credential store as described [here](http://knox.apache.org/books/knox-0-6-0/user-guide.html#Special+note+on+parameter+main.ldapRealm.contextFactory.systemPassword).  
Note2: This search base should constrain the search as much as possible to limit the amount of data returned by the query.

To try and start things off on the right foot, lets try and execute an LDAP bind against AD.
For this you will need your values for Server Host, Server Port, System Username and System Password described in the table above.

This initial testing will be done using command line tools from [OpenLDAP][openldap].
If you don't have these command line tools available don't despair Knox provides some alternatives I'll show you later. 

The command below will help ensure that the values for Server Host, Server Port, System Username and System Password are correct.  

```sh
ldapwhoami -h ad-nano.qe.hortonworks.com -p 389 -x -D 'CN=Kevin Minder,CN=Users,DC=hwqe,DC=hortonworks,DC=com' -w 'p@ssw0rd'
```
- -h: Server Host
- -p: Server Port
- -D: System Username
- -w: System Password

In this case I'm using my own test account as the system user because it happens to have search privileges.
For me this returns the output below.

```
u:HWQE\kminder
```

Now lets make sure that the system user can actually search.
Note that in this case the system user is searching for itself because -D and -b use the same value.
You could change -b to search for other users.

```sh
ldapsearch -h ad-nano.qe.hortonworks.com -p 389 -x -D 'CN=Kevin Minder,CN=Users,DC=hwqe,DC=hortonworks,DC=com' -w 'p@ssw0rd' -b 'CN=Kevin Minder,CN=Users,DC=hwqe,DC=hortonworks,DC=com'
```
- -b: System Username 

This should return all of the LDAP attributes for the system user.
Take note of a few key attributes like objectClass which here is "person", and sAMAccountName which here is "kminder".

```
# extended LDIF
#
# LDAPv3
# base <CN=Users,DC=hwqe,DC=hortonworks,DC=com> with scope subtree
# filter: CN=Kevin Minder
# requesting: ALL
#

# Kevin Minder, Users, hwqe.hortonworks.com
dn: CN=Kevin Minder,CN=Users,DC=hwqe,DC=hortonworks,DC=com
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: Kevin Minder
sn: Minder
givenName: Kevin
distinguishedName: CN=Kevin Minder,CN=Users,DC=hwqe,DC=hortonworks,DC=com
instanceType: 4
whenCreated: 20151117175833.0Z
whenChanged: 20151117175919.0Z
displayName: Kevin Minder
uSNCreated: 26688014
uSNChanged: 26688531
name: Kevin Minder
objectGUID:: Eedvw9dqoUK/ERLNEFrQ5w==
userAccountControl: 66048
badPwdCount: 0
codePage: 0
countryCode: 0
badPasswordTime: 130922583862610479
lastLogoff: 0
lastLogon: 130922584014955481
pwdLastSet: 130922567133848037
primaryGroupID: 513
objectSid:: AQUAAAAAAAUVAAAA7TkHmDQ43l1xd4O/MigBAA==
accountExpires: 9223372036854775807
logonCount: 0
sAMAccountName: kminder
sAMAccountType: 805306368
userPrincipalName: kminder@hwqe.hortonworks.com
objectCategory: CN=Person,CN=Schema,CN=Configuration,DC=hwqe,DC=hortonworks,DC=com
dSCorePropagationData: 16010101000000.0Z
lastLogonTimestamp: 130922567243691894

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
```

Next lets check the values for Search Base, Search Attribute and Search Class with a command like the one below.  
Again, don't forget to scroll right.

```sh
ldapsearch -h ad-nano.qe.hortonworks.com -p 389 -x -D 'CN=Kevin Minder,CN=Users,DC=hwqe,DC=hortonworks,DC=com' -w 'p@ssw0rd' -b 'CN=Users,DC=hwqe,DC=hortonworks,DC=com' -z 5 '(objectClass=person)' sAMAccountName
```
- -z 5: Limit the search results to 5 entries. Note that by default AD will only return a max of 1000 entries.
- '(objectClass=person)': Limit the search results to entries where objectClass=person
- sAMAccountName: Return only the SAMAccountName attribute
 
If no results were returned go back and check the output from the search above for corrected settings.
The results for this command should look something like what is shown below.
Take note of the various attribute values returned for sAMAccountName.

```
# extended LDIF
#
# LDAPv3
# base <CN=Users,DC=hwqe,DC=hortonworks,DC=com> with scope subtree
# filter: (objectClass=person)
# requesting: sAMAccountName
#

# Administrator, Users, hwqe.hortonworks.com
dn: CN=Administrator,CN=Users,DC=hwqe,DC=hortonworks,DC=com
sAMAccountName: Administrator

# guest, Users, hwqe.hortonworks.com
dn: CN=guest,CN=Users,DC=hwqe,DC=hortonworks,DC=com
sAMAccountName: guest

# cloudbase-init, Users, hwqe.hortonworks.com
dn: CN=cloudbase-init,CN=Users,DC=hwqe,DC=hortonworks,DC=com
sAMAccountName: cloudbase-init

# krbtgt, Users, hwqe.hortonworks.com
dn: CN=krbtgt,CN=Users,DC=hwqe,DC=hortonworks,DC=com
sAMAccountName: krbtgt

# ambari-server, Users, hwqe.hortonworks.com
dn: CN=ambari-server,CN=Users,DC=hwqe,DC=hortonworks,DC=com
sAMAccountName: ambari-server

# search result
search: 2
result: 4 Size limit exceeded

# numResponses: 6
# numEntries: 5
```

# Sample 3

Since you have verified all of the environmental information required for authentication, you are ready to create your third topology file.
Just as with the first example, this first one will only include authentication.
We will tackle authorization later.
Create this sample3 topology file.
Take care to replace all of the example environment values with the correct values for your environment.

| Parameter                                    | Example                                                     |
| -------------------------------------------- | ----------------------------------------------------------- |
| main.ldapRealm                               | org.apache.hadoop.gateway.shirorealm.KnoxLdapRealm          |
| main.ldapContextFactory                      | org.apache.hadoop.gateway.shirorealm.KnoxLdapContextFactory | 
| main.ldapRealm.contextFactory                | $ldapContextFactory                                         |
| main.ldapRealm.contextFactory.url            | ldap://ad-nano.qe.hortonworks.com:389                       |
| main.ldapRealm.contextFactory.systemUsername | CN=Kevin Minder,CN=Users,DC=hwqe,DC=hortonworks,DC=com      |
| main.ldapRealm.contextFactory.systemPassword | p@ssw0rd                                                    |
| main.ldapRealm.searchBase                    | CN=Users,DC=hwqe,DC=hortonworks,DC=com                      |
| main.ldapRealm.userSearchAttributeName       | sAMAccountName                                              |
| main.ldapRealm.userObjectClass               | person                                                      |
| urls./**                                     | authcBasic                                                  |
<br>

[\<GATEWAY_HOME>/conf/topologies/sample3.xml](/static/activedirectory/sample3.xml)

```xml
<topology>
  <gateway>

    <provider>
      <role>authentication</role>
      <name>ShiroProvider</name>
      <enabled>true</enabled>
      <param name="main.ldapRealm" value="org.apache.hadoop.gateway.shirorealm.KnoxLdapRealm"/>
      <param name="main.ldapContextFactory" value="org.apache.hadoop.gateway.shirorealm.KnoxLdapContextFactory"/>
      <param name="main.ldapRealm.contextFactory" value="$ldapContextFactory"/>

      <param name="main.ldapRealm.contextFactory.url" value="ldap://ad-nano.qe.hortonworks.com:389"/>
      <param name="main.ldapRealm.contextFactory.systemUsername" value="CN=Kevin Minder,CN=Users,DC=hwqe,DC=hortonworks,DC=com"/>
      <param name="main.ldapRealm.contextFactory.systemPassword" value="p@ssw0rd"/>

      <param name="main.ldapRealm.searchBase" value="CN=Users,DC=hwqe,DC=hortonworks,DC=com"/>
      <param name="main.ldapRealm.userSearchAttributeName" value="sAMAccountName"/>
      <param name="main.ldapRealm.userObjectClass" value="person"/>

      <param name="urls./**" value="authcBasic"/>
    </provider>

  </gateway>
  <service>
    <role>KNOX</role>
  </service>
</topology>
```

We could go straight to trying to access the Knox Admin API with cURL as we did before.
However, lets take this opportunity to explore the new LDAP diagnostic tools introduced in Apache Knox 0.7.0.

This first command helps diagnose basic connectivity and system user issues. 

```sh
bin/knoxcli.sh system-user-auth-test --cluster sample3
System LDAP Bind successful.
```

If the command above works you can move on to testing the LDAP search configuration settings of the topology.
If you don't provide the username and password via the command line switches you will be prompted to enter them. 

```sh
bin/knoxcli.sh user-auth-test --cluster sample3 --u kminder --p p@ssw0rd
LDAP authentication successful!
```

Once all of that is working go ahead and try the cURL command.

```sh
curl -u kminder:p@ssw0rd -ik 'https://localhost:8443/gateway/sample3/api/v1/version'
```

# Sample 4

The next step is to enable authorization.
To accomplish this there is a bit more environmental information needed.
The OpenLDAP command line tools are useful here again to ensure those we have the correct values.
Authorization requires group lookup and for this the TODO

```sh
~/Projects/knox-devguide-tests/install/knox-0.7.0-SNAPSHOT> ldapsearch -h ad-nano.qe.hortonworks.com -p 389 -x -D 'CN=Kevin Minder,CN=Users,DC=hwqe,DC=hortonworks,DC=com' -w 'p@ssw0rd' -b 'OU=groups,DC=hwqe,DC=hortonworks,DC=com' -z 2
# extended LDIF
#
# LDAPv3
# base <OU=groups,DC=hwqe,DC=hortonworks,DC=com> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# groups, hwqe.hortonworks.com
dn: OU=groups,DC=hwqe,DC=hortonworks,DC=com
objectClass: top
objectClass: organizationalUnit
ou: groups
distinguishedName: OU=groups,DC=hwqe,DC=hortonworks,DC=com
instanceType: 4
whenCreated: 20150812202242.0Z
whenChanged: 20150812202242.0Z
uSNCreated: 42340
uSNChanged: 42341
name: groups
objectGUID:: RYIcbNyVWki5HmeANfzAbA==
objectCategory: CN=Organizational-Unit,CN=Schema,CN=Configuration,DC=hwqe,DC=h
 ortonworks,DC=com
dSCorePropagationData: 20150827225949.0Z
dSCorePropagationData: 20150812202242.0Z
dSCorePropagationData: 16010101000001.0Z

# scientist, groups, hwqe.hortonworks.com
dn: CN=scientist,OU=groups,DC=hwqe,DC=hortonworks,DC=com
objectClass: top
objectClass: group
cn: scientist
member: CN=sam repl2,CN=Users,DC=hwqe,DC=hortonworks,DC=com
member: CN=sam repl1,CN=Users,DC=hwqe,DC=hortonworks,DC=com
member: CN=sam repl,CN=Users,DC=hwqe,DC=hortonworks,DC=com
member: CN=bob,CN=Users,DC=hwqe,DC=hortonworks,DC=com
member: CN=sam,CN=Users,DC=hwqe,DC=hortonworks,DC=com
distinguishedName: CN=scientist,OU=groups,DC=hwqe,DC=hortonworks,DC=com
instanceType: 4
whenCreated: 20150812213414.0Z
whenChanged: 20150828231624.0Z
uSNCreated: 42355
uSNChanged: 751045
name: scientist
objectGUID:: iXhbVo7kJUGkiQ+Sjlm0Qw==
objectSid:: AQUAAAAAAAUVAAAA7TkHmDQ43l1xd4O/SgUAAA==
sAMAccountName: scientist
sAMAccountType: 536870912
groupType: -2147483644
objectCategory: CN=Group,CN=Schema,CN=Configuration,DC=hwqe,DC=hortonworks,DC=
 com
dSCorePropagationData: 20150827225949.0Z
dSCorePropagationData: 16010101000001.0Z

# search result
search: 2
result: 4 Size limit exceeded

# numResponses: 3
# numEntries: 2
```

```sh
ldapsearch -h ad-nano.qe.hortonworks.com -p 389 -x -D 'CN=Kevin Minder,CN=Users,DC=hwqe,DC=hortonworks,DC=com' -w 'p@ssw0rd' -b 'OU=groups,DC=hwqe,DC=hortonworks,DC=com' member
# extended LDIF
#
# LDAPv3
# base <OU=groups,DC=hwqe,DC=hortonworks,DC=com> with scope subtree
# filter: (objectclass=*)
# requesting: member
#

# groups, hwqe.hortonworks.com
dn: OU=groups,DC=hwqe,DC=hortonworks,DC=com

# scientist, groups, hwqe.hortonworks.com
dn: CN=scientist,OU=groups,DC=hwqe,DC=hortonworks,DC=com
member: CN=sam repl2,CN=Users,DC=hwqe,DC=hortonworks,DC=com
member: CN=sam repl1,CN=Users,DC=hwqe,DC=hortonworks,DC=com
member: CN=sam repl,CN=Users,DC=hwqe,DC=hortonworks,DC=com
member: CN=bob,CN=Users,DC=hwqe,DC=hortonworks,DC=com
member: CN=sam,CN=Users,DC=hwqe,DC=hortonworks,DC=com

# analyst, groups, hwqe.hortonworks.com
dn: CN=analyst,OU=groups,DC=hwqe,DC=hortonworks,DC=com
member: CN=testLdap1,CN=Users,DC=hwqe,DC=hortonworks,DC=com
member: CN=sam repl2,CN=Users,DC=hwqe,DC=hortonworks,DC=com
member: CN=sam repl1,CN=Users,DC=hwqe,DC=hortonworks,DC=com
member: CN=sam repl,CN=Users,DC=hwqe,DC=hortonworks,DC=com
member: CN=bob,CN=Users,DC=hwqe,DC=hortonworks,DC=com
member: CN=tom,CN=Users,DC=hwqe,DC=hortonworks,DC=com
member: CN=sam,CN=Users,DC=hwqe,DC=hortonworks,DC=com

# knox_hdp_users, groups, hwqe.hortonworks.com
dn: CN=knox_hdp_users,OU=groups,DC=hwqe,DC=hortonworks,DC=com
member: CN=sam repl2,CN=Users,DC=hwqe,DC=hortonworks,DC=com
member: CN=sam repl1,CN=Users,DC=hwqe,DC=hortonworks,DC=com
member: CN=sam repl,CN=Users,DC=hwqe,DC=hortonworks,DC=com
member: CN=sam,CN=Users,DC=hwqe,DC=hortonworks,DC=com

# knox_no_users, groups, hwqe.hortonworks.com
dn: CN=knox_no_users,OU=groups,DC=hwqe,DC=hortonworks,DC=com

# test grp, groups, hwqe.hortonworks.com
dn: CN=test grp,OU=groups,DC=hwqe,DC=hortonworks,DC=com
member: CN=testLdap1,CN=Users,DC=hwqe,DC=hortonworks,DC=com
member: CN=sam,CN=Users,DC=hwqe,DC=hortonworks,DC=com

# search result
search: 2
result: 0 Success

# numResponses: 7
# numEntries: 6
```

| Paramete                            | Description | Example                                 |
| ----------------------------------- | ----------- | --------------------------------------- |
| main.ldapRealm.userSearchBase       |             | CN=Users,DC=hwqe,DC=hortonworks,DC=com  | 
| main.ldapRealm.authorizationEnabled |             | true                                    |
| main.ldapRealm.groupSearchBase      |             | OU=groups,DC=hwqe,DC=hortonworks,DC=com |
| main.ldapRealm.groupObjectClass     |             | group                                   |
| main.ldapRealm.groupIdAttribute     |             | CN                                      |
| main.ldapRealm.memberAttribute      |             | member                                  |
<br>

[\<GATEWAY_HOME>/conf/topologies/sample4.xml](/static/activedirectory/sample4.xml)

```xml
<topology>
    <gateway>

        <provider>
            <role>authentication</role>
            <name>ShiroProvider</name>
            <enabled>true</enabled>
            <param name="main.ldapRealm" value="org.apache.hadoop.gateway.shirorealm.KnoxLdapRealm"/>
            <param name="main.ldapContextFactory" value="org.apache.hadoop.gateway.shirorealm.KnoxLdapContextFactory"/>
            <param name="main.ldapRealm.contextFactory" value="$ldapContextFactory"/>

            <param name="main.ldapRealm.contextFactory.url" value="ldap://ad-nano.qe.hortonworks.com:389"/>
            <param name="main.ldapRealm.contextFactory.systemUsername" value="CN=Kevin Minder,CN=Users,DC=hwqe,DC=hortonworks,DC=com"/>
            <param name="main.ldapRealm.contextFactory.systemPassword" value="p@ssw0rd"/>

            <param name="main.ldapRealm.userSearchBase" value="CN=Users,DC=hwqe,DC=hortonworks,DC=com"/>
            <param name="main.ldapRealm.userSearchAttributeName" value="sAMAccountName"/>
            <param name="main.ldapRealm.userObjectClass" value="person"/>

            <param name="main.ldapRealm.authorizationEnabled" value="true"/>
            <param name="main.ldapRealm.groupSearchBase" value="OU=groups,DC=hwqe,DC=hortonworks,DC=com"/>
            <param name="main.ldapRealm.groupObjectClass" value="group"/>
            <param name="main.ldapRealm.groupIdAttribute" value="CN"/>
            <param name="main.ldapRealm.memberAttribute" value="member"/>

            <param name="urls./**" value="authcBasic"/>
        </provider>

    </gateway>
    <service>
        <role>KNOX</role>
    </service>
</topology>
```

```sh
bin/knoxcli.sh user-auth-test --cluster sample4 --u sam --p 'Horton!#works' --g
LDAP authentication successful!
sam is a member of: analyst
sam is a member of: knox_hdp_users
sam is a member of: test grp
sam is a member of: scientist
```

```sh
bin/knoxcli.sh user-auth-test --cluster sample4 --u kminder --p 'p@ssw0rd' --g
LDAP authentication successful!
kminder does not belong to any groups
Warn: main.ldapGroupContextFactory is not present in topology
Warn: main.ldapRealm.memberAttributeValueTemplate is not present in topology
You were looking for this user's groups but this user does not belong to any.
Your topology file may be incorrectly configured for group lookup.
```

# Sample 5

[\<GATEWAY_HOME>/conf/topologies/sample5.xml](/static/activedirectory/sample5.xml)

```xml
<topolgy>
    <gateway>

        <provider>
            <role>authentication</role>
            <name>ShiroProvider</name>
            <enabled>true</enabled>
            <param name="main.ldapRealm" value="org.apache.hadoop.gateway.shirorealm.KnoxLdapRealm"/>
            <param name="main.ldapContextFactory" value="org.apache.hadoop.gateway.shirorealm.KnoxLdapContextFactory"/>
            <param name="main.ldapRealm.contextFactory" value="$ldapContextFactory"/>
            <param name="main.ldapRealm.contextFactory.url" value="ldap://ad-nano.qe.hortonworks.com:389"/>
            <param name="main.ldapRealm.contextFactory.systemUsername" value="CN=Kevin Minder,CN=Users,DC=hwqe,DC=hortonworks,DC=com"/>
            <param name="main.ldapRealm.contextFactory.systemPassword" value="p@ssw0rd"/>
            <param name="main.ldapRealm.userSearchBase" value="CN=Users,DC=hwqe,DC=hortonworks,DC=com"/>
            <param name="main.ldapRealm.userSearchAttributeName" value="sAMAccountName"/>
            <param name="main.ldapRealm.userObjectClass" value="person"/>
            <param name="main.ldapRealm.authorizationEnabled" value="true"/>
            <param name="main.ldapRealm.groupSearchBase" value="OU=groups,DC=hwqe,DC=hortonworks,DC=com"/>
            <param name="main.ldapRealm.groupObjectClass" value="group"/>
            <param name="main.ldapRealm.groupIdAttribute" value="CN"/>
            <param name="main.ldapRealm.memberAttribute" value="member"/>
            <param name="urls./**" value="authcBasic"/>
        </provider>

        <provider>
            <role>authorization</role>
            <name>AclsAuthz</name>
            <enabled>true</enabled>
            <param name="knox.acl" value="*;knox_hdp_users;*"/>
        </provider>

    </gateway>
    <service>
        <role>KNOX</role>
    </service>
</topology>
```

```sh
curl -u kminder:p@ssw0rd -ik 'https://localhost:8443/gateway/sample5/api/v1/version'
403
```

```sh
curl -u sam:'Horton!#works' -ik 'https://localhost:8443/gateway/sample5/api/v1/version'
200
```

# Sample 6

Next lets enable caching because out of the box this important performance enhancement isn't enabled.
TODO

| Parameter                                   | Description | Example                                       |
| ------------------------------------------- | ----------- | --------------------------------------------- |
| main.cacheManager                           |             | org.apache.shiro.cache.ehcache.EhCacheManager | 
| main.securityManager.cacheManager           |             | $cacheManager                                 |
| main.ldapRealm.authenticationCachingEnabled |             | true                                          |
<br>


[\<GATEWAY_HOME>/conf/topologies/sample6.xml](/static/activedirectory/sample6.xml)

```xml
<topology>
    <gateway>

        <provider>
            <role>authentication</role>
            <name>ShiroProvider</name>
            <enabled>true</enabled>
            <param name="main.ldapRealm" value="org.apache.hadoop.gateway.shirorealm.KnoxLdapRealm"/>
            <param name="main.ldapContextFactory" value="org.apache.hadoop.gateway.shirorealm.KnoxLdapContextFactory"/>
            <param name="main.ldapRealm.contextFactory" value="$ldapContextFactory"/>
            <param name="main.ldapRealm.contextFactory.url" value="ldap://ad-nano.qe.hortonworks.com:389"/>
            <param name="main.ldapRealm.contextFactory.systemUsername" value="CN=Kevin Minder,CN=Users,DC=hwqe,DC=hortonworks,DC=com"/>
            <param name="main.ldapRealm.contextFactory.systemPassword" value="p@ssw0rd"/>
            <param name="main.ldapRealm.userSearchBase" value="CN=Users,DC=hwqe,DC=hortonworks,DC=com"/>
            <param name="main.ldapRealm.userSearchAttributeName" value="sAMAccountName"/>
            <param name="main.ldapRealm.userObjectClass" value="person"/>
            <param name="main.ldapRealm.authorizationEnabled" value="true"/>
            <param name="main.ldapRealm.groupSearchBase" value="OU=groups,DC=hwqe,DC=hortonworks,DC=com"/>
            <param name="main.ldapRealm.groupObjectClass" value="group"/>
            <param name="main.ldapRealm.groupIdAttribute" value="CN"/>
            <param name="main.ldapRealm.memberAttribute" value="member"/>

            <param name="main.cacheManager" value="org.apache.shiro.cache.ehcache.EhCacheManager"/>
            <param name="main.securityManager.cacheManager" value="$cacheManager"/>
            <param name="main.ldapRealm.authenticationCachingEnabled" value="true"/>

            <param name="urls./**" value="authcBasic"/>
        </provider>

        <provider>
            <role>authorization</role>
            <name>AclsAuthz</name>
            <enabled>true</enabled>
            <param name="knox.acl" value="*;knox_hdp_users;*"/>
        </provider>

    </gateway>
    <service>
        <role>KNOX</role>
    </service>
</topology>
```

```sh
curl -u sam:'Horton!#works' -ik 'https://localhost:8443/gateway/sample6/api/v1/version'
```

Now unplug your network cable, turn off Wifi or disconnect from VPN.
The intent being to temporarily prevent access to the ActiveDirectory server.
The command below will continue to work even though no cookies are used and the ActiveDirectory server cannot be contacted.
This is because the invocation above caused the user's authentication and authorization information to be cached.

```sh
curl -u sam:'Horton!#works' -ik 'https://localhost:8443/gateway/sample6/api/v1/version'
```

The command below uses and invalid password and is intended to prove that the previously authenticated credentials are re-verified.
It is important to note that Knox does not store the actual password in the cache for this verification but rather a one way hash of the password.

```sh
curl -u sam:invalid-password -ik 'https://localhost:8443/gateway/sample6/api/v1/version'
```

# Sample 7

Finally lets put it all together in a real topology file that doesn't use the Knox Admin API.
The important things to observe here are:

1. the host and ports for the Hadoop services will need to be changed to match your environment
1. the inclusion of the Hadoop services instead of the Knox Admin API
1. the inclusion of the identity-assertion provider
1. the exclusion of the hostmap provider as this is rarely required unless running Hadoop on local VMs with port mapping
 
[\<GATEWAY_HOME>/conf/topologies/sample7.xml](/static/activedirectory/sample7.xml)

```xml
<topology>
    <gateway>

        <provider>
            <role>authentication</role>
            <name>ShiroProvider</name>
            <enabled>true</enabled>
            <param name="main.ldapRealm" value="org.apache.hadoop.gateway.shirorealm.KnoxLdapRealm"/>
            <param name="main.ldapContextFactory" value="org.apache.hadoop.gateway.shirorealm.KnoxLdapContextFactory"/>
            <param name="main.ldapRealm.contextFactory" value="$ldapContextFactory"/>

            <param name="main.ldapRealm.contextFactory.url" value="ldap://ad-nano.qe.hortonworks.com:389"/>
            <param name="main.ldapRealm.contextFactory.systemUsername" value="CN=Kevin Minder,CN=Users,DC=hwqe,DC=hortonworks,DC=com"/>
            <param name="main.ldapRealm.contextFactory.systemPassword" value="p@ssw0rd"/>

            <param name="main.ldapRealm.userSearchBase" value="CN=Users,DC=hwqe,DC=hortonworks,DC=com"/>
            <param name="main.ldapRealm.userSearchAttributeName" value="sAMAccountName"/>
            <param name="main.ldapRealm.userObjectClass" value="person"/>

            <param name="main.ldapRealm.authorizationEnabled" value="true"/>
            <param name="main.ldapRealm.groupSearchBase" value="OU=groups,DC=hwqe,DC=hortonworks,DC=com"/>
            <param name="main.ldapRealm.groupObjectClass" value="group"/>
            <param name="main.ldapRealm.groupIdAttribute" value="CN"/>
            <param name="main.ldapRealm.memberAttribute" value="member"/>

            <param name="main.cacheManager" value="org.apache.shiro.cache.ehcache.EhCacheManager"/>
            <param name="main.securityManager.cacheManager" value="$cacheManager"/>
            <param name="main.ldapRealm.authenticationCachingEnabled" value="true"/>

            <param name="urls./**" value="authcBasic"/>
        </provider>

        <provider>
            <role>authorization</role>
            <name>AclsAuthz</name>
            <enabled>true</enabled>
            <param name="knox.acl" value="*;knox_hdp_users;*"/>
        </provider>

        <provider>
            <role>identity-assertion</role>
            <name>Default</name>
            <enabled>true</enabled>
        </provider>

    </gateway>

    <service>
        <role>NAMENODE</role>
        <url>hdfs://your-nn-host.your-domain.com:8020</url>
    </service>

    <service>
        <role>JOBTRACKER</role>
        <url>rpc://your-jt-host.your-domain.com:8050</url>
    </service>

    <service>
        <role>WEBHDFS</role>
        <url>http://your-nn-host.your-domain.com:50070/webhdfs</url>
    </service>

    <service>
        <role>WEBHCAT</role>
        <url>http://your-webhcat-host.your-domain.com:50111/templeton</url>
    </service>

    <service>
        <role>OOZIE</role>
        <url>http://your-oozie-host.your-domain.com:11000/oozie</url>
    </service>

    <service>
        <role>WEBHBASE</role>
        <url>http://your-hbase-host.your-domain.com:60080</url>
    </service>

    <service>
        <role>HIVE</role>
        <url>http://your-hive-host.your-domain.com:10001/cliservice</url>
    </service>

    <service>
        <role>RESOURCEMANAGER</role>
        <url>http://your-rm-host.your-domain.com:8088/ws</url>
    </service>

</topology>
```

[knox-site]: http://knox.apache.org/
[knox-lists]: http://knox.apache.org/mail-lists.html
[user-guide]: http://knox.apache.org/books/knox-0-6-0/user-guide.html "Apache Knox User's Guide"
[dev-guide]: http://knox.apache.org/books/knox-0-6-0/dev-guide.html "Apache Knox Developer's Guide"
[apache-ds]: https://directory.apache.org/apacheds/
[openldap]: http://www.openldap.org/