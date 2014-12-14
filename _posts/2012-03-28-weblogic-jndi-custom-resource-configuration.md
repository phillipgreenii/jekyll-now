---
layout: post
title: WebLogic JNDI Custom Resource Configuration
date: '2012-03-28T13:22:00.001-04:00'
author: Phillip Green II
tags:
- java
- ldap
- jndi
- weblogic
modified_time: '2012-04-10T09:54:00.000-04:00'
blogger_id: tag:blogger.com,1999:blog-3096944600800047027.post-6713929924113472892
blogger_orig_url: http://coder-in-training.blogspot.com/2012/03/weblogic-jndi-custom-resource.html
---

I have been using WebLogic for a couple years now.
While it is better than Oracle Application Server, it is missing some very important functionality.
Today's complaint is the lacking of a way to add custom JNDI resources.
This was discussed at stackoverflow: [Custom resource in JNDI on different application servers][custom-jndi-resource].
I am currently using Oracle WebLogic 10g (10.3.5), so my solution may not be relevant in future versions.

I was very frustrated that other application servers add the functionality, but not WebLogic. The best solution I found was this: [weblogic-jndi-startup][]. I tried it and it works, but it has some limitations. The objects added must have a constructor that accepts a single String parameter. What I want is to add Properties and perhaps LDAP connections (similar to JDBC connections).

#weblogic-jndi-custom-resource-configuration

I drew from [Roger][roger-parkinson-googlecode]'s code and made this project: [weblogic-jndi-custom-resource-configuration][]. It provides a couple separate Initializers: `StringInitializer`, `PropertiesInitializer`, and `LdapDirContextInitializer`. Installation and configuration instructions can be found at [weblogic-jndi-custom-resource-configuration][]. Essentially, you copy the [jar](https://bitbucket.org/phillip_green_idmworks/weblogic-jndi-custom-resource-configuration/downloads/weblogic-jndi-custom-resource-configuration-1.0.jar) to the domain classpath of WebLogic and add Initializers as Startup Classes.

##StringInitializer

`StringInitializer` will load a java.lang.String instance into JNDI at a specified location.

###Configuration

![StringInitializer Configuration][img-server-node-config]

|Field|Value|
|:----|:----|
|Name|server-node|
|Class Name|	**com.idmworks.weblogic.jndiconfiguration.StringInitializer**|
|Arguments|	**server/node=Test**|
|Failure is Fatal|	unticked|
|Run Before Application Deployments|	unticked|
|Run Before Application Activations|	**ticked**|

The magic happens with the arguments: `server/node=Test`. `StringInitializer` will insert the value, "Test", at the JNDI location `server/node`.

###Usage

####Retrieving String from JNDI

```java
final InitialContext initialContext = new InitialContext();
final String node = (String) initialContext.lookup("server/node");
```

In previous example, the `String` instance is available just as any other object in JNDI.

##PropertiesInitializer

`PropertiesInitializer` will load a `java.util.Properties` instance (based on a properties file) into JNDI at a specified location.

###Configuration

![PropertiesInitializer Configuration][img-myapp-prop-config]

|Field|Value|
|:----|:----|
|Name|myapp-properties|
|Class Name|	**com.idmworks.weblogic.jndiconfiguration.PropertiesInitializer**|
|Arguments|	**properties/myapp=/path/to/myapp.properties**|
|Failure is Fatal|	unticked|
|Run Before Application Deployments|	unticked|
|Run Before Application Activations|	**ticked**|

As before, we focus on the arguments: `properties/myapp=/path/to/myapp.properties`. `PropertiesInitializer` will create an instance of `java.util.Properties` from the properties found at `/path/to/myapp.properties`. It next places it at the JNDI location `properties/myapp`.

###Usage

####Retrieving Properties from JNDI

```java
final InitialContext initialContext = new InitialContext();
final Properties myappProperties = (Properties) initialContext.lookup("properties/myapp");
```

In previous example, the `Properties` instance is available just as any other object in JNDI.

##LdapDirContextInitializer

`LdapDirContextInitializer` will load a `java.util.Properties` instance (based on a properties file).
It adds into JNDI a `javax.naming.Reference` with a `javax.naming.spi.ObjectFactory` that will create a `DirContext`.
The created `DirContext` from the factory will be configured by the specified properties file.

###Configuration

![LdapDirContextInitializer Configuration][img-ldap-test-config]

|Field|Value|
|:----|:----|
|Name|ldap-test|
|Class Name|	**com.idmworks.weblogic.jndiconfiguration.LdapDirContextInitializer**|
|Arguments|	**ldap/test=/path/to/ldap.properties**|
|Failure is Fatal|	unticked|
|Run Before Application Deployments|	unticked|
|Run Before Application Activations|	**ticked**|



####/path/to/ldap.properties
```properties
java.naming.provider.url=ldap://localhost:389/dc=home
#The following lines could be uncommented if the LDAP Connection requires authentication
#java.naming.security.principal=cn=user,dc=home
#java.naming.security.credentials=password
```

The arguments: `ldap/test=/path/to/ldap.properties` work similar to `PropertiesInitializer`.
`LdapDirContextInitializer` will create an instance of `java.util.Properties` from the properties found at `/path/to/ldap.properties`.
What is added at the JNDI location `ldap/test` is a `javax.naming.Reference`.
The reference is configured with a `javax.naming.spi.ObjectFactory` that will generate a new `DirContext` with the properties specified at ``/path/to/ldap.properties`.
Currently, the properties is also stored in JNDI at `ldap/test__properties`.

###Usage

####Retrieving LDAP Connection from JNDI
```java
final InitialContext initialContext = new InitialContext();
final DirContext ldapContext = (DirContext) initialContext.lookup("ldap/test");
```

In previous example, the `DirContext` instance is available just as any other object in JNDI.
Each time `initialContext.lookup("ldap/test")`` is called, a new `DirContext` is created, so it is the responsibility of the caller to close the connection.

##References
 * [Roger Parkinson's Page on Googlecode][roger-parkinson-googlecode]
 * [Custom resource in JNDI on different application servers.][custom-jndi-resource]
 * [weblogic-jndi-startup][]
 * [weblogic-jndi-custom-resource-configuration][]


[roger-parkinson-googlecode]: <https://code.google.com/u/roger.parkinson35/> "Roger Parkinson's Page on Googlecode"
[custom-jndi-resource]: <http://stackoverflow.com/questions/3749799/custom-resource-in-jndi-on-different-application-servers> "Custom resource in JNDI on different application servers."
[weblogic-jndi-startup]: <http://code.google.com/p/weblogic-jndi-startup/> "weblogic-jndi-startup"
[weblogic-jndi-custom-resource-configuration]: <https://github.com/phillipgreenii/weblogic-jndi-custom-resource-configuration> "weblogic-jndi-custom-resource-configuration"


[img-server-node-config]: <{{ site.baseurl }}/images/weblogic-jndi-custom-resource-configuration/StringInitializer-server-node-configuration.png> "server node Configuration"
[img-myapp-prop-config]: <{{ site.baseurl }}/images/weblogic-jndi-custom-resource-configuration/PropertiesInitializer-myapp-properties-configuration.png> "myapp properties Configuration"
[img-ldap-test-config]: <{{ site.baseurl }}/images/weblogic-jndi-custom-resource-configuration/LdapDirContextInitializer-ldap-test-configuration.png> "LDAP Test Configuration"
