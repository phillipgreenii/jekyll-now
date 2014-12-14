---
layout: post
title: Mapped Diagnostic Context (MDC) in Weblogic 10.3.5
date: '2012-03-05T10:03:00.001-05:00'
author: Phillip Green II
tags:
- java
- java.util.logging
- JUL
- log4j
- logging
- MDC
- SLF4J
- weblogic
modified_time: '2012-04-01T09:28:46.748-04:00'
blogger_id: tag:blogger.com,1999:blog-3096944600800047027.post-2797122452442648911
blogger_orig_url: http://coder-in-training.blogspot.com/2012/03/mapped-diagnostic-context-mdc-in.html
---

##Problem: Mapped Diagnostic Context (MDC) values aren't available in Weblogic 10.3.5 Logs

I am developing a web application that is deployed on Weblogic 10.3.5 and I am using [SLF4J][slf4j] for logging. I used the Mapped Diagnositc Context ([MDC][slf4j-mdc]) feature of SLF4J to include the current user and the current remote IP for each log message.

```text
Log message from logback showing MDC values (IP and UserName)
[2012-03-02 12:56:29,770] INFO  ii.green.phillip.ImportantClass - IP[127.0.0.1] - UserName[pdgreen] - Info Message
```

I use a servlet filter to set the MDC values and everything worked just fine when I used log4j or logback as the SLF4J implementation. While the logs look fine with a file appender, they don't show up in the Weblogic logs when I use the console appender.

##Attempt 1: Switch to java.util.logging (JUL) for SLF4J implementation

By default, Weblogic is configured to log with [JUL][jul]. So, the easiest change I could make to my web application was to switch the SLF4J from logback to JUL. After the change, all of my logs showed up in Weblogic. The problems is that JUL does not support MDC, so my logs weren't showing the MDC values.

```text
Log message from Weblogic using JUL showing message without MDC values
<Mar 3, 2012 7:56:29 PM EST> <Info> <Server> <BEA-000000> <Info Message>
```

##Attempt 2: Switch Weblogic to use log4j
Weblogic can be configured to use JUL or [log4j][]. Since log4j supports MDC, I was hoping this would fix my problem. I followed the steps to configure Weblogic to use log4j. I created the following `log4j.xml`:

####log4j.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE log4j:configuration SYSTEM "log4j.dtd" >
<log4j:configuration xmlns:log4j="http://jakarta.apache.org/log4j/">
<appender name="console" class="org.apache.log4j.ConsoleAppender"/>
<root>
<priority value="info" />
<appender-ref ref="console" />
</root>
</log4j:configuration>
```
I tried moving `log4j.xml` around, but was not able to find the appropriate location and I kept seeing the following error:

```text
Errors I was not able to fix with log4j configuration for weblogic
log4j:WARN No appenders could be found for logger (ii.green.phillip.ImportantClass).
log4j:WARN Please initialize the log4j system properly.
log4j:WARN See http://logging.apache.org/log4j/1.2/faq.html#noconfig for more info.
```
Due to the previous error, none of my log messages ever showed up in Weblogic.

##Attempt 3: Enable stdout and stderr logging in Weblogic
While trying the previous attempts, I noticed the following tick boxes in Weblogic:

![Weblogic Logging Configuration][img-weblogic-stdout-stderr-screenshot]


They can be found at `Environment -> Servers -> ${servername} -> Logging -> Advanced`.

First, I reset everything back to the original configuration: Weblogic back to using JUL and my app using logback. Next, I enabled the `Redirect stdout logging` and `Redirect stderr logging`. Finally, I configured two console appenders in logback so that `INFO` messages are logged through stdout and anything `WARN` or above is logged through stderr. I left out the exact configuration because it didn't work as I wanted it to. The logs did show up, just not quite right:

```text
'Redirect stdout logging' and 'Redirect stderr logging' log everything at the same level
<Mar 1, 2012 4:58:54 PM EST> <Notice> <Stdout> <BEA-000000> <INFO: Info Message>
<Mar 1, 2012 4:58:55 PM EST> <Notice> <StdErr> <BEA-000000> <WARN: Warn Message>
<Mar 1, 2012 4:58:56 PM EST> <Notice> <StdErr> <BEA-000000> <ERROR: Error Message>
```

If you look at the logs I was seeing, everything is logged as "Notice". I can distinguish stdout and stderr, but not by using levels. Having different levels is important because I am not the person monitoring the logs in Weblogic. They need to see `ERROR`/`SEVERE` or `WARN`/`WARNING` level messages. I disabled `Redirect stdout logging` and `Redirect stderr logging` and went back to the drawing board.

##Final Attempt: LogbackToJulAppender
After the last attempt, I was more familiar with logback and realized that I could make a custom appender that would take my logback log messages and route them through JUL. This is different from using the JUL implemenation for SLF4J. What I needed to do was have logback create the message portion of the log before it is routed into JUL.

####ii.green.phillip.LogbackToJulAppender
```java
package ii.green.phillip;

import ch.qos.logback.classic.jul.JULHelper;
import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.classic.spi.IThrowableProxy;
import ch.qos.logback.classic.spi.ThrowableProxy;
import ch.qos.logback.core.AppenderBase;
import ch.qos.logback.core.Layout;
import java.util.logging.LogRecord;

/**
* Logback Appender that outputs logs to java.util.Logging.

* This is different from slf4j-jdk14-*.jar because the message generation
* is handled by SLF4J so things like MDC are supported.
* Code was augmented from org.slf4j.impl.JDK14LoggerAdapter
*
* @author pdgreen
*/
public class LogbackToJulAppender extends AppenderBase<ILoggingEvent> {

  private Layout<ILoggingEvent> layout;

  public Layout<ILoggingEvent> getLayout() {
    return layout;
  }

  public void setLayout(Layout<ILoggingEvent> layout) {
    this.layout = layout;
  }

  @Override
  public void start() {
    if (this.layout == null) {
      addError("No layout set for the appender named [" + name + "].");
      return;
    }
    super.start();
  }

  @Override
  protected void append(ILoggingEvent eventObject) {
    final String message = layout.doLayout(eventObject);

    final LogRecord record = new LogRecord(JULHelper.asJULLevel(eventObject.getLevel()), message);
    record.setLoggerName(eventObject.getLoggerName());
    record.setThrown(retrieveThrowable(eventObject.getThrowableProxy()));

    fillCallerData(eventObject, record);

    JULHelper.asJULLogger(eventObject.getLoggerName()).log(record);
  }

  Throwable retrieveThrowable(final IThrowableProxy iThrowableProxy) {
    if (iThrowableProxy instanceof ThrowableProxy) {
      return ((ThrowableProxy) iThrowableProxy).getThrowable();
    }
    return null;
  }

  final private void fillCallerData(ILoggingEvent eventObject, LogRecord record) {
    final StackTraceElement[] steArray = eventObject.getCallerData();
    if (steArray.length > 0) {
      //I think this is correct
      final StackTraceElement ste = steArray[0];
      // setting the class name has the side effect of setting
      // the needToInferCaller variable to false.
      record.setSourceClassName(ste.getClassName());
      record.setSourceMethodName(ste.getMethodName());
    }
  }
}
```

####logback.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <property name="DEFAULT_PATTERN" value="[%d{ISO8601}] [%thread] %-5level %logger{36} - IP[%X{ip}] - UserName[%X{username}] - %msg%n" />
  <!-- Pattern for redirected logs to java.util.logging for weblogic
  the exception is handled by java.util.logging, so
  %nopex is needed or else the exception will be printed twice -->
  <property name="WEBLOGIC_PATTERN" value="IP[%X{ip}] - UserName[%X{username}] - %msg%nopex" />

  <!-- Application log file -->
  <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>webconsole.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
      <fileNamePattern>webconsole.%d{yyyy-ww}.zip</fileNamePattern>
      <maxHistory>6</maxHistory>
    </rollingPolicy>
    <encoder>
      <pattern>${DEFAULT_PATTERN}</pattern>
    </encoder>
  </appender>

  <!-- This will redirect INFO and about into weblogic so that the logs can be viewed in the Weblogic Console -->
  <appender name="WEBLOGIC" class="ii.green.phillip.LogbackToJulAppender">
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
      <level>INFO</level>
    </filter>
    <layout class="ch.qos.logback.classic.PatternLayout">
      <pattern>${WEBLOGIC_PATTERN}</pattern>
    </layout>
  </appender>

  <root level="info">
    <appender-ref ref="FILE" />
    <appender-ref ref="WEBLOGIC" />
  </root>
</configuration>
```

My application now has two appenders. The first is a `RollingFileAppender` that logs everything through logback with all of its glory. The second is the custom appender that only redirect logs of level `INFO` and above into JUL for Weblogic. Now the application admins can see errors in the Weblogic console.

```text
Log message from Weblogic using JUL and LogbackToJulAppender showing message wit MDC values
<Mar 3, 2012 7:56:29 PM EST> <Info> <Server> <BEA-000000> <IP[127.0.0.1] - UserName[pdgreen] - Info Message>
```

[slf4j]: <http://www.slf4j.org/> "SLF4J"
[slf4j-mdc]: <http://www.slf4j.org/manual.html#mdc> "SLF4J MDC Documentation"
[jul]: <http://docs.oracle.com/javase/6/docs/api/java/util/logging/package-summary.html> "JavaDocs for java.util.logging"
[log4j]: <http://logging.apache.org/log4j/1.2/index.html> "log4j"

[img-weblogic-stdout-stderr-screenshot]: <{{ site.baseurl }}/images/mapped-diagnostic-context-mdc-in/screenshot-weblogic-10.3.5-stdout-stderr-redirect.png> "Weblogic 10.3.5 stdout-stderr redirect screenshot"
