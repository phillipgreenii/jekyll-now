---
layout: post
title: Specifying ProjectStage in JNDI with WebLogic
date: '2012-04-03T15:13:00.000-04:00'
author: Phillip Green II
tags:
- java
- production
- development
- projectstage
- jndi
- weblogic
- jsf
modified_time: '2012-04-03T15:13:00.152-04:00'
blogger_id: tag:blogger.com,1999:blog-3096944600800047027.post-1829775392878379031
blogger_orig_url: http://coder-in-training.blogspot.com/2012/04/specifying-projectstage-in-jndi-with.html
---
##Overview
Starting in JSF 2.0 ([JSR-314][jsr314]) a feature was added called [ProjectStage][projectstage].
This is an attribute that signifies the stage where the project is running: `Development`, `UnitTest`, `SystemTest`, or `Production`.
The idea is that if the code knows where it is running, it can plan better.
As an example, in a production environment certain optimizations could occur during startup that would cause better overal performance. However, these optimizations during start up is a problem during development because during development there is a lot more shutting down and starting up of the server.

There are two ways to specify the ProjectStage: `web.xml` and JNDI.
The problem with using `web.xml` is that you need to update or override it as you go move from one environment to the next.
JNDI is configured on the application server, so it is the ideal place to configure ProjectStage.

In WebLogic (11g), you cannot add custom resources to JNDI.
So, out of the box, you can not specify ProjectStage in JNDI.
In a previous [post][weblogic-jndi-cr-config-post], I described
[Weblogic JNDI Custom Resource Configuration][weblogic-jndi-cr-config-source], which is a project where I configure JNDI in WebLogic using Startup Classes.

##Adding ProjectStage to JNDI
Follow the documentation at [Weblogic JNDI Custom Resource Configuration][weblogic-jndi-cr-config-source] to install it.
Use the following configuration:

![ProjectStage Configuration][img-jsf-stage-config]

|Property|Value|
|----|------|
|Name|jsf-stage|
|Class Name:|**com.idmworks.weblogic.jndiconfiguration.StringInitializer**|
|Failure is Fatal:|unticked|
|Run Before Application Deployments:|unticked|
|Run Before Application Activations:|**ticked**|

This will configure `jsf/master/ProjectStage` to store the value of "Development"


###Add Resource to web.xml
In the previous step, "Development" was added to the global name space as `jsf/master/ProjectStage`.
The JSF JSR Documentation states that it looks for the ProjectStage at `java:comp/env/jsf/ProjectStage` which is the local namespace.
In order to define it in the local name space, we need to specify it in `web.xml`.

```xml
<web-app version="2.5" xmlns:web="http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://java.sun.com/xml/ns/javaee" xsi:schemalocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd">
  <!-- ... other configuration removed ... -->
  <resource-ref>
    <res-ref-name>jsf/ProjectStage</res-ref-name>
    <res-type>java.lang.String</res-type>
    <res-auth>Container</res-auth>
  </resource-ref>
  <!-- ... other configuration removed ... -->
</web-app>
```


###Add Resource to weblogic.xml
In the previous step, we allocated the place holder for `java:comp/env/jsf/ProjectStage`, we now need to map the global location to the local place holder:

####weblogic.xml
```xml
<weblogic-web-app xmlns:j2ee="http://java.sun.com/xml/ns/j2ee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://www.bea.com/ns/weblogic/90" xsi:schemalocation="http://www.bea.com/ns/weblogic/90 http://www.bea.com/ns/weblogic/90/weblogic-web-app.xsd">
  <!-- ... other configuration removed ... -->
  <resource-description>
    <res-ref-name>jsf/ProjectStage</res-ref-name>
    <jndi-name>jsf/master/ProjectStage</jndi-name>
  </resource-description>
</weblogic-web-app>
```

###Verify ProjectStage
Below is a small snippet of code that you can add to any facelet to check what the current projectStage is set to

```jsp
Stage: #{facesContext.application.projectStage}
```


##References
 * [JSR-000314 JavaServer™ Faces 2.0][jsr314]
 * [JSF 2.0 New Feature Preview Series (Part 1): ProjectStage][projectstage]
 * [WebLogic JNDI Custom Resource Configuration][weblogic-jndi-cr-config-post]
 * [Github: WebLogic JNDI Custom Resource Configuration Source Code][weblogic-jndi-cr-config-source]
 * [What is resource-ref in web.xml used for?][resource-ref-used-for]

[jsr314]: <http://jcp.org/aboutJava/communityprocess/final/jsr314/index.html> "JSR-000314 JavaServer™ Faces 2.0"
[projectstage]: <https://blogs.oracle.com/rlubke/entry/jsf_2_0_new_feature2> "JSF 2.0 New Feature Preview Series (Part 1): ProjectStage"
[weblogic-jndi-cr-config-post]: <http://coder-in-training.blogspot.com/2012/03/weblogic-jndi-custom-resource.html> "WebLogic JNDI Custom Resource Configuration"
[weblogic-jndi-cr-config-source]: <https://github.com/phillipgreenii/weblogic-jndi-custom-resource-configuration> "Github: WebLogic JNDI Custom Resource Configuration Source Code"
[resource-ref-used-for]: <http://stackoverflow.com/questions/2887967/what-is-resource-ref-in-web-xml-used-for> "What is resource-ref in web.xml used for?"


[img-jsf-stage-config]: <{{ site.baseurl }}/images/specifying-projectstage-in-jndi-with/StringInitializer-jsf-stage-configuration.png> "ProjectStage Configuration"
