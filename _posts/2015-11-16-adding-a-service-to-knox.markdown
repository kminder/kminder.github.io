---
layout: post
title:  "Adding a service to Apache Knox"
date:   2015-11-16
categories: knox
---

This article will cover adding a service to [Apache Knox][knox-site].
The intention here is to provide an intentionally simple example, avoiding complexity wherever possible.
The goal being getting something working as a starting point upon which more complicated scenarios could build.
You may want to review the Apache Knox [User's Guide][user-guide] and [Developer's Guide][dev-guide] before reading this. 

The API used here is an [OpenWeatherMap API](http://openweathermap.org/current#zip) that returns the current weather information for a given zip code.
This is the cURL command to access this API directly.  Give it a try.

```sh
curl 'http://api.openweathermap.org/data/2.5/weather?zip=95054,us&appid=2de143494c0b295cca9337e1e96b00e0'
```

This should return a JSON similar to the output shown below.
Your results probably won't be nicely formatted.
Note that I'm not giving anything away here with the `appid`.
This is what they use in all of their [examples](http://openweathermap.org/current#zip).

```json
{  
   "coord":{"lon":-121.98,"lat":37.43},
   "weather":[{"id":800,"main":"Clear","description":"Sky is Clear","icon":"01d"}],
   "base":"cmc stations",
   "main":{"temp":282.235,"pressure":998.58,"humidity":51,"temp_min":282.235,"temp_max":282.235,"sea_level":1038.24,"grnd_level":998.58},
   "wind":{"speed":4.92,"deg":347.5},
   "clouds":{"all":0},
   "dt":1447699480,
   "sys":{"message":0.0049,"country":"US","sunrise":1447685347,"sunset":1447721770},
   "id":5323631,
   "name":"Alviso",
   "cod":200
}
```

This is the cURL command showing how we will expose that service via the gateway.
Don't try this now, it won't work until later!

```sh
curl -ku guest:guest-password 'https://localhost:8443/gateway/sandbox/weather/data/2.5/weather?zip=95054,us&appid=2de143494c0b295cca9337e1e96b00e0'
```

So the fundamental job of the gateway is to translate the effective request URL it receives to the target URL and then transfer the request and response bodies.
In this example we will ignore the request and response bodies and focus on the request URL.
Lets take a look at how these two request URLs are related.

| Source  |                            URL                                  | 
| ------- | --------------------------------------------------------------- | 
| Gateway | https://localhost:8443/gateway/sandbox/weather/data/2.5/weather |
| Direct  | http://api.openweathermap.org/data/2.5/weather                  |

<br/>
We can start by breaking down the Gateway URL and understanding where each of the URL parts come from.
 
| Part             | Details                                                                           |
| -----------------| --------------------------------------------------------------------------------- |
| https            | The gateway has SSL/TLS enabled: See ssl.enabled in gateway-site.xml              |
| localhost        | The gateway is listening on 0.0.0.0: See gateway.host in gateway-site.xml         |
| 8443             | The gateway is listening on port 8443: See gateway.port in gateway-site.xml       |
| gateway          | The gateway context path is 'gateway': See gateway.path in gateway-site.xml       |
| sandbox          | The topology file that includes the WEATHER service is named sandbox.xml          |
| weather          | The unique root of all WEATHER service URLs.  Identified in service's service.xml |
| data/2.5/weather | This portion of the URL is handled by the service's rewrite.xml rules             |

<br/>
In contrast we really only care about two parts of the service's Direct URL.

| Part                          | Details                                      |
| ------------------------------| -------------------------------------------- |
| http://api.openweathermap.org | The network address of the service itself.   |
| data/2.5/weather              | The path for the weather API of the service. |

<br/>

Now we need to get down to the business of actually making the gateway proxy this service.
To do that we will be using the new configuration based extension model introduced in Knox 0.6.0.
That will involve adding two new files under the \<GATEWAY_HOME\>/data/services directory and then modifying a topology file.
 
Note: The \<GATEWAY_HOME\> here represents the directory where Apache Knox is installed.

First you need to create a directory to hold your new service definition files.
There are two conventions at work here that ultimately (but only loosely) relate to the content of the service.xml it will contain.
Below the `<GATEWAY_HOME>/data/services` directory you will need to create a parent and child directory `weather/0.0.1`.
As a convention the names of these directories duplicate the values in the attributes of the root element of the contained service.xml.

Create the two files with the content shown below and place them in the directories indicated.
The links also provide the files for your convenience.


[\<GATEWAY_HOME\>/data/services/weather/0.0.1/service.xml](/static/weather/service.xml)

```xml
<service role="WEATHER" name="weather" version="0.0.1">
  <routes>
    <route path="/weather/**"/>
  </routes>
</service>
```

[\<GATEWAY_HOME\>/data/services/weather/0.0.1/rewrite.xml](/static/weather/rewrite.xml)

```xml
<rules>
  <rule dir="IN" name="WEATHER/weather/inbound" pattern="*://*:*/**/weather/{path=**}?{**}">
    <rewrite template="{$serviceUrl[WEATHER]}/{path=**}?{**}"/>
  </rule>
</rules>
```

Once that is complete, the topology file must be updated to activate this new service in the runtime.
In this case the sandbox.xml topology file is used but you may have another topology file such as default.xml.
Edit which ever topology file you prefer and add the <service>...</service> markup shown below.
If you aren't using sandbox.xml be careful to replace sandbox with the name of your topology file through these examples.

\<GATEWAY_HOME\>/conf/topologies/sandbox.xml

```xml
<topology>
  ...
  <service>
    <role>WEATHER</role>
    <url>http://api.openweathermap.org:80</url>
  </service>
</topology>
```

With all of these changes made you must restart your Knox gateway server.
Often times this isn't necessary but adding a new service definition under [\<GATEWAY_HOME\>/data/services requires restart.
 
You should now be able to execute the curl command from way back at the top that accesses the OpenWeatherMap API via the gateway.
 
```sh
curl -ku guest:guest-password 'https://localhost:8443/gateway/sandbox/weather/data/2.5/weather?zip=95054,us&appid=2de143494c0b295cca9337e1e96b00e0'
```

Now that the new service definition is working lets go back and connect all the dots.
This should help take some of the mystery out of the configuration above.
The most important and confusing aspect is how values in different files are interrelated.
I will focus on that.

# service.xml
The service.xml file defines the high level URL patterns that will be exposed by the gateway for a service.
If you are getting HTTP 404 errors there is probably a problem with this configuration.

`<service role="WEATHER"`

- The role/implementation/version triad is used through Knox for integration plugins.
- Think of the role as an interface in Java.
- This attribute declares what role this service "implements".
- This will need to match the topology file's <topology><service><role> for this service. 

`<service name="weather"`

- In the role/implementation/version triad this is the implementation.
- Think of this as a Java implementation class name relative to an interface.
- As a matter of convention this should match the directory beneath \<GATEWAY_HOME>/data/services
- The topology file can optionally contain \<topology>\<service>\<name> but usually doesn't.
This would be used to select a specific implementation of a role if there were multiple. 

`<service version="0.0.1"`

- As a matter of convention this should match the directory beneath the service implementation name.
- The topology file can optionally contain \<topology>\<service>\<version> but usually doesn't.
This would be used to select a specific version of an implementation there were multiple.
This can be important if the protocols for a service evolve over time.

`<service><routes><route path="/weather/**"`

- This tells the gateway that all requests starting starting with /weather/ are handled by this service.
- Due to a limitation this will not include requests to /weather (i.e. no trailing /)
- The `**` means zero or more paths similar to Ant.
- The scheme, host, port, gateway and topology components are not included (e.g. https://localhost:8443/gateway/sandbox) 
- Routes can, but typically don't, take query parameters into account.
- In this simple form there is _no direct relationship_ between the route path and the rewrite rules! 

# rewrite.xml
The rewrite.xml is configuration that drives the rewrite provider within Knox.
It is important to understand that at runtime for a given topology, all of the rewrite.xml files for all active services are combined into a single file.
This explains some of the seemingly complex patterns and naming conventions. 

`<rules><rule dir="IN"`

- Here dir means direction and IN means it should apply to a request.
- This rule is a global rule meaning that any other service can request that a URL be rewritten as they process URLs.
The rewrite provider keeps distinct trees of URL patterns for IN and OUT rules so that services can be specific about which to apply.
- If it were not global it would not have a direction and probably not a pattern in the <rule> element. 

`<rules><rule name="WEATHER/weather/inbound"`

- Rules can be explicitly invoked in various ways.  In order to allow that they are named.
- The convention is role/name/\<service specific hierarchy>.
- Remember that all rules share a single namespace.

`<rules><rule pattern="*://*:*/**/weather/{path=**}?{**}"`

- Defines the URL pattern for which this rule will apply.
- The * matches exactly one segment of the URL.
- The ** matches zero or more segments of the URL.
- The {path=**} matches zero or more path segments and provides access them as a parameter named 'path'.
- The {**} matches zero or more query parameters and provides access to them by name.
- The values from matched {...} segments are "consumed" by the rewrite template below.

`<rules><rule><rewrite template="{$serviceUrl[WEATHER]}/{path=**}?{**}"`

- Defines how the URL matched by the rule will be rewritten.
- The $serviceUrl[WEATHER]} looks up the \<service>\<url> for the \<service>\<role>WEATHER. 
This is a implemented as rewrite function and is another custom extension point.
- The {path=**} extracts zero or more values for the 'path' parameter from the matched URL.
- The {**} extracts any "unused" parameters and uses them as query parameters.

# sandbox.xml
 
`<topology><service><role>WEATHER`

- This causes the service definition with role WEATHER to be loaded into the runtime.
- Since <name> and <verion> are not present a default is selected if there are multiple options.

`<topology><service><url>http://api.openweathermap.org:80`

- This populates the data used by {$serviceUrl[WEATHER]} in the rules with the correct target URL.
  
Hopefully all of this provides a more gentle introduction to adding a service to Apache Knox than might be offered in the [Apache Knox Developer's Guide][dev-guide].
If you have more questions, comments or suggestions please feel free so joing the community at [Apache Knox][knox-site].
In particular you might be interested in one of the [mailing lists][knox-lists].

[knox-site]: http://knox.apache.org/
[knox-lists]: http://knox.apache.org/mail-lists.html
[user-guide]: http://knox.apache.org/books/knox-0-6-0/user-guide.html "Apache Knox User's Guide"
[dev-guide]: http://knox.apache.org/books/knox-0-6-0/dev-guide.html "Apache Knox Developer's Guide"
