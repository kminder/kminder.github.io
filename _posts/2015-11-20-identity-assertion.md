---
layout: post
title:  "Adding an Identity Assertion Provider to Apache Knox"
date:   2015-11-20
categories: knox
---

This article covers adding an identity asserter to [Apache Knox][knox-site].

In particular, the goal of this article are to highlight the code and steps necessary to build and install an identity asserter provider into Apache Knox.
The larger role of the identity assertion provider within Knox is covered in the [Knox User's Guide][usr-guide].
For our purposes, the identity assertion provider shown will translate group names retrieved from LDAP, which happen to be upper case, to lower case.
This can be used to solve a common problem caused by the importing of groups into various external systems. 

Please setup an environment as described in '[Setting up Apache Knox in three easy steps](/knox/2015/11/18/setting-up-knox.html)'.
The examples here assume the environment described there as a base environment.

My college Larry McCay implemented all the code for this article in his [github repo](https://github.com/lmccay/gateway-provider-identity-assertion-groupstolower).

```sh
~/Projects> git clone https://github.com/lmccay/gateway-provider-identity-assertion-groupstolower.git
~/Projects> cd gateway-provider-identity-assertion-groupstolower
```

The code below is the business of translating the groups names to lower case.
It fairly straight forwardly queries for a specific type of principals from the current subject, lower cases the name and stores the result in a string array.
For purpose built identity assertion filters the logic really can be this simple.

[GroupsToLowerAssertionFilter.java](https://github.com/lmccay/gateway-provider-identity-assertion-groupstolower/blob/master/src/main/java/org/apache/hadoop/gateway/identityasserter/GroupsToLowerAssertionFilter.java)

```java
public class GroupsToLowerAssertionFilter extends CommonIdentityAssertionFilter {
  
  @Override
  public String[] mapGroupPrincipals( String mappedPrincipalName, Subject subject ) {
    ArrayList<String> groups = new ArrayList<String>();
    Set<GroupPrincipal> currentGroups = subject.getPrincipals( GroupPrincipal.class );
    for ( GroupPrincipal group : currentGroups ) {
      groups.add( group.getName().toLowerCase() );
    }
    return groups.toArray( new String[groups.size()] );
  }

}
```

The only other class in the project is the ServiceLoader class used by Knox to find and load the plugin.
Notice how the role of "identity-assertion" and name of "GroupsToLower" match the topology file shown farther below.
The contributeFilter method is invoked as the topology file is processed to create the WAR and filter chains that are used at runtime.
Here the method is simply adding the filter class above to the filter chain for the resource.

[GroupsToLowerAssertionContributor.java](https://github.com/lmccay/gateway-provider-identity-assertion-groupstolower/blob/master/src/main/java/org/apache/hadoop/gateway/identityasserter/GroupsToLowerAssertionContributor.java)

```java
public class GroupsToLowerAssertionContributor extends ProviderDeploymentContributorBase {

  private static final String ROLE = "identity-assertion";
  private static final String NAME = "GroupsToLower";
  private static final String FILTER = GroupsToLowerAssertionFilter.class.getName();

  public String getRole() {
    return ROLE;
  }

  public String getName() {
    return NAME;
  }

  public void contributeFilter(
      DeploymentContext context, Provider provider, Service service,
      ResourceDescriptor resource, List<FilterParamDescriptor> params ) {

    resource.addFilter().name( getName() ).role( getRole() ).impl( FILTER ).params( params );

  }

}
```

The Maven pom.xml file is setup to build a JAR than can be installed to know via the 'package' phase.

```sh
~/Projects/gateway-provider-identity-assertion-groupstolower> mvn package 
```

Installing the plugin is as simple as copying the JAR to the 'ext' directory in \<GATEWAY_HOME>.
In the example below, The directory '~/Projects/knox/install/knox-0.7.0-SNAPSHOT' is \<GATEWAY_HOME>.  

```sh
~/Projects/gateway-provider-identity-assertion-groupstolower> cp target/gateway-provider-identity-assertion-groupstolower-0.0.1.jar ~/Projects/knox/install/knox-0.7.0-SNAPSHOT/ext  
```

Since you essentially just modified the classpath of the Knox server you will need to stop the server if you have it running.
We are also going to change the user information in the demo LDAP server so that needs to be stopped as well.

```sh
~/Projects/gateway-provider-identity-assertion-groupstolower> cd ~/Projects/knox
~/Projects/knox> ant stop-test-servers
```

Replace the "out of the box" \<GATEWAY_HOME>/conf/users.ldif file with the one linked below.

[\<GATEWAY_HOME>/conf/users.ldif](/static/identity-assertion/users.ldif)

Take note of the group definitions in that users.ldif file as they have been setup specifically for this scenaior.
The important attribute is `cn` of each of the group entities.  For example `cn:ADMINS` of `cn=admins,ou=groups,dc=hadoop,dc=apache,dc=org`.
These are intentionally capitalized to allow for the demonstration of the GroupsToLower identity assertion filter.
An excerpt of the full file is shown below.

```
# Sample user: admin
dn: uid=admin,ou=people,dc=hadoop,dc=apache,dc=org
objectclass:top
objectclass:person
objectclass:organizationalPerson
objectclass:inetOrgPerson
cn: Admin
sn: Admin
uid: admin
userPassword:admin-password

# Sample group: ADMINS
dn: cn=admins,ou=groups,dc=hadoop,dc=apache,dc=org
objectclass:top
objectclass: groupofnames
cn: ADMINS
description: admin group
member: uid=admin,ou=people,dc=hadoop,dc=apache,dc=org
```

Next a topology file that incorporate the new identity assertion needs to be created.
Create the file linked below as \<GATEWAY_HOME>/conf/topologies/tolower.xml. 

[\<GATEWAY_HOME>/conf/topologies/tolower.xml](/static/identity-assertion/tolower.xml)

An excerpt of that file is shown below.
The first provider definition shows how the \<role>identity-assertion\</role> and \<name>GroupsToLower\</name> cause the new implementation to be included.
The second provider definition for role authorization is interesting because it uses the lower case role `admins` in its configuration.

```xml
<provider>
    <role>identity-assertion</role>
    <name>GroupsToLower</name>
    <enabled>true</enabled>
</provider>

<provider>
    <role>authorization</role>
    <name>AclsAuthz</name>
    <enabled>true</enabled>
    <param name="KNOX.acl" value="*;admins;*"/>
</provider>
```

Now you can start the Knox servers and everything it ready to test.

```sh
~/Projects/knox> ant start-test-servers
```

Use the cURL command below to access the Knox Admin API.
This will cause both an authentication and authorization to occur.

```sh
~/Projects/knox> curl -ku admin:admin-password 'https://localhost:8443/gateway/tolower/api/v1/version'
<?xml version="1.0" encoding="UTF-8"?>
<ServerVersion>
   <version>0.7.0-SNAPSHOT</version>
   <hash>9632b697060bfeffa2e03425451a3e9b3980c45e</hash>
</ServerVersion>
```

Take a look at the audit log to verify that the upper case group name ADMINs from the users.ldif file was indeed mapped to lower case.
Also the request would have failed authorization if this had not been the case.

```sh
~/Projects/knox> tail -6 install/knox-0.7.0-SNAPSHOT/logs/gateway-audit.log
15/11/20 14:15:55 ||0b9118cf-ae86-4127-8bb7-33c608018f9e|audit|KNOX||||access|uri|/gateway/tolower/api/v1/version|unavailable|Request method: GET
15/11/20 14:15:55 ||0b9118cf-ae86-4127-8bb7-33c608018f9e|audit|KNOX|admin|||authentication|uri|/gateway/tolower/api/v1/version|success|
15/11/20 14:15:55 ||0b9118cf-ae86-4127-8bb7-33c608018f9e|audit|KNOX|admin|||authentication|uri|/gateway/tolower/api/v1/version|success|Groups: [ADMINS]
15/11/20 14:15:55 ||0b9118cf-ae86-4127-8bb7-33c608018f9e|audit|KNOX|admin|||identity-mapping|principal|admin|success|Groups: [admins]
15/11/20 14:15:55 ||0b9118cf-ae86-4127-8bb7-33c608018f9e|audit|KNOX|admin|||authorization|uri|/gateway/tolower/api/v1/version|success|
15/11/20 14:15:56 ||0b9118cf-ae86-4127-8bb7-33c608018f9e|audit|KNOX|admin|||access|uri|/gateway/tolower/api/v1/version|success|Response status: 200
```

Hopefully this article provides a quick guide to creating and installing identity assertion providers for Apache Knox.
There is more information in both the Apache Knox [User's Guide][usr-guide] and [Developer's Guide][dev-guide].
If you have more questions, comments or suggestions please join the [Apache Knox][knox-site] community.
In particular you might be interested in one of the [mailing lists][knox-lists].

[knox-site]: http://knox.apache.org/
[knox-lists]: http://knox.apache.org/mail-lists.html
[usr-guide]: http://knox.apache.org/books/knox-0-6-0/user-guide.html "Apache Knox User's Guide"
[dev-guide]: http://knox.apache.org/books/knox-0-6-0/dev-guide.html "Apache Knox Developer's Guide"
